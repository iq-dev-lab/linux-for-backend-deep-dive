# 03. 소켓 옵션 — TCP_NODELAY부터 SO_REUSEPORT까지

## 🎯 핵심 질문

- Nagle 알고리즘이 Redis 레이턴시에 어떤 영향을 미치는가?
- `SO_KEEPALIVE`는 커널이 유휴 연결을 어떻게 감지하는가?
- `SO_REUSEPORT`로 여러 프로세스가 같은 포트를 공유할 때 커널의 부하 분산 방식은?
- `SO_LINGER`를 0으로 설정하면 어떤 일이 일어나는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Redis 클라이언트가 pipelining 없이 요청을 보낼 때 레이턴시가 불규칙하다.** Nagle 알고리즘이 작은 패킷을 모아 보내려 하기 때문이다. Redis 서버는 `TCP_NODELAY`를 설정해 모든 응답을 즉시 전송하지만, 클라이언트 라이브러리도 `TCP_NODELAY`를 설정해야 한다. Jedis, Lettuce 모두 기본적으로 `TCP_NODELAY`를 활성화한다.

**nginx upstream 서버로의 연결이 죽어있는데 nginx가 모른다.** 네트워크 장애 후 한쪽이 모르는 "half-open" 연결 상태다. `SO_KEEPALIVE`를 설정하면 커널이 주기적으로 탐지 패킷을 보내 연결 상태를 확인한다. nginx의 `keepalive_timeout`, `keepalive_requests` 설정과 OS의 `tcp_keepalive_*` 파라미터가 조합된다.

**Tomcat 워커 프로세스가 여러 개인데 Accept 경합이 발생한다.** 모든 워커가 하나의 `listen()` fd로 `accept()`를 호출하면 "thundering herd" 문제가 생긴다. `SO_REUSEPORT`로 각 워커가 독립 소켓을 가지면 이 경합이 없어진다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```java
// 실수 1: TCP_NODELAY 미설정으로 레이턴시 불규칙
Socket s = new Socket("redis-host", 6379);
// Nagle 알고리즘 활성화 상태
// 작은 Redis 명령어(수십 바이트)가 버퍼링됨 → 최대 200ms 지연 가능

// 올바른 설정:
s.setTcpNoDelay(true);  // TCP_NODELAY 활성화

// 실수 2: SO_KEEPALIVE 없이 장기 연결 유지
// 방화벽이 비활성 연결을 60초 후 자동 제거
// 애플리케이션은 연결이 살아있다고 생각하고 write() 시도
// → TCP RST 수신 → "Connection reset by peer"

// 실수 3: SO_LINGER=0으로 TIME_WAIT 제거 시도
// Socket s = ...;
// s.setSoLinger(true, 0);  // RST로 강제 종료
// → TIME_WAIT 없앨 수 있지만 데이터 유실 위험
// → 상대방이 ECONNRESET 오류 수신
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 소켓 옵션 상태 확인
$ ss -toni  # -o: timer 정보, -i: TCP 내부 정보
State    Recv-Q  Send-Q
ESTAB    0       0       10.0.0.1:8080  10.0.0.2:54321
  timer:(keepalive,7min,0)  ← SO_KEEPALIVE 활성화, 7분 후 probe
  cubic wscale:9,7 ...
  nonagle  ← TCP_NODELAY 활성화됨

# SO_KEEPALIVE 커널 파라미터
$ sysctl net.ipv4.tcp_keepalive_time     # 비활성 후 첫 probe까지 (기본 7200초 = 2시간)
$ sysctl net.ipv4.tcp_keepalive_intvl    # probe 간격 (기본 75초)
$ sysctl net.ipv4.tcp_keepalive_probes   # probe 횟수 (기본 9회)
# → 2시간 + 75초 * 9 = 약 2시간 11분 후에야 죽은 연결 감지!

# 실용적인 SO_KEEPALIVE 설정 (5분 이내 감지)
$ sysctl -w net.ipv4.tcp_keepalive_time=300     # 5분
$ sysctl -w net.ipv4.tcp_keepalive_intvl=10     # 10초
$ sysctl -w net.ipv4.tcp_keepalive_probes=3     # 3회
# → 5분 + 10초 * 3 = 5분 30초 이내 감지
```

---

## 🔬 내부 동작 원리

### TCP_NODELAY와 Nagle 알고리즘

```
Nagle 알고리즘 (RFC 896):
  목적: 작은 패킷의 과도한 전송 방지 (네트워크 효율)

  규칙: 다음 조건 중 하나를 만족할 때만 패킷 전송
    ① 데이터 크기 >= MSS (Maximum Segment Size, ~1448 bytes)
    ② 이전 전송에 대한 ACK 수신
    ③ 타이머 만료 (최대 200ms)

Nagle 알고리즘이 문제가 되는 경우:

  Redis PING 명령어 (4 bytes) 전송 시:
    t=0ms: send("PING\r\n") → 4바이트, Nagle 버퍼링
    t=0ms: 서버가 직전 응답에 대한 ACK 미수신 상태 (delayed ACK)
    → 200ms 타이머 대기 후 전송!

  Delayed ACK (서버측):
    서버가 ACK를 즉시 안 보내고 최대 200ms 대기 후 전송
    (ACK에 데이터를 함께 실어 효율화)

  Nagle + Delayed ACK 조합:
    클라이언트: Nagle로 200ms 버퍼링
    서버: Delayed ACK로 200ms 대기
    → 최악의 경우 400ms 지연 발생!

TCP_NODELAY 설정:
  Nagle 알고리즘 비활성화
  → 모든 write()가 즉시 패킷으로 전송
  → 레이턴시 감소, 패킷 수 증가

setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof(one));

적합한 경우: Redis, gRPC, 게임 서버 (레이턴시 우선)
부적합한 경우: 파일 전송, 벌크 데이터 (처리량 우선)
```

### SO_KEEPALIVE 동작

```
SO_KEEPALIVE 활성화 시 커널 동작:

  tcp_keepalive_time 동안 데이터 교환 없음
        ↓
  keepalive probe 전송 (TCP 빈 ACK 패킷)
        ↓
  응답 있음: 연결 살아있음, 타이머 리셋
  응답 없음: tcp_keepalive_intvl 간격으로 추가 probe
        ↓
  tcp_keepalive_probes 회 연속 응답 없음
        ↓
  커널이 소켓 오류 표시 (ETIMEDOUT)
  → 다음 read()/write() 시 애플리케이션에 오류 반환

setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &one, sizeof(one));
// 필요 시 개별 소켓별 파라미터 조정:
setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &idle, sizeof(idle));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &intvl, sizeof(intvl));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &cnt, sizeof(cnt));

주의: SO_KEEPALIVE는 연결 "유지"가 아닌 "감지"
  → 방화벽이 비활성 연결을 끊으려면 keepalive 주기 < 방화벽 타임아웃
  → AWS ELB 기본 idle timeout: 60초
     tcp_keepalive_time 기본값: 7200초 (2시간) → 훨씬 큼!
  → 실용적 설정: tcp_keepalive_time = 60 (방화벽 타임아웃 이내)
```

### SO_REUSEPORT 동작

```
SO_REUSEPORT 없이 (전통적 방식):
  프로세스 A  ←─ accept() ─  ┐
  프로세스 B  ←─ accept() ─  ├─  listen socket (포트 80)
  프로세스 C  ←─ accept() ─  ┘
  
  → 여러 프로세스가 하나의 Accept Queue에 경합
  → thundering herd 문제: 이벤트 발생 시 모두 깨어났다가 하나만 성공
  → mutex 경합으로 오버헤드

SO_REUSEPORT 설정 시:
  프로세스 A  ←─  listen socket A (포트 80)  ┐
  프로세스 B  ←─  listen socket B (포트 80)  ├─ 커널 부하 분산
  프로세스 C  ←─  listen socket C (포트 80)  ┘

  커널: 새 연결 → 해시 기반으로 어느 소켓으로 분산할지 결정
  → 각 소켓 독립적인 Accept Queue
  → 경합 없음, 분산 처리

setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &one, sizeof(one));

Nginx SO_REUSEPORT:
  # nginx.conf
  events {
      use epoll;
  }
  http {
      server {
          listen 80 reuseport;  ← SO_REUSEPORT 활성화
      }
  }
  → worker 프로세스마다 독립 소켓
  → 처리량 향상, Accept 경합 제거

주의: Rolling Restart 시 연결 손실 가능
  → 이전 worker 종료 시 해당 소켓의 대기 연결도 함께 종료
```

### SO_LINGER — 종료 시 데이터 처리

```
close() 기본 동작 (SO_LINGER 미설정):
  close() 호출 → 즉시 반환
  커널이 백그라운드에서 남은 데이터 전송 완료
  → TIME_WAIT 상태로 진입 (2 * MSL 유지)

SO_LINGER with l_linger > 0:
  struct linger sl = { .l_onoff=1, .l_linger=5 };
  setsockopt(fd, SOL_SOCKET, SO_LINGER, &sl, sizeof(sl));

  close() 호출 → 최대 5초 대기 (전송 완료 시 즉시 반환)
  5초 이내 전송 완료: 정상 종료
  5초 초과: RST 전송 (데이터 유실 가능)
  → 버퍼 데이터 전송 보장하지만 차단됨

SO_LINGER with l_linger = 0 (즉시 RST):
  struct linger sl = { .l_onoff=1, .l_linger=0 };
  
  close() 호출 → 즉시 RST 전송
  → TIME_WAIT 없음!
  → 버퍼의 미전송 데이터 유실
  → 상대방 ECONNRESET 오류

사용 시나리오:
  정상: 기본 설정 (데이터 보장, TIME_WAIT 허용)
  부하 테스트: l_linger=0 (TIME_WAIT 없앰, 포트 재사용)
  절대 사용 금지: 프로덕션 API 서버 (데이터 유실)
```

---

## 💻 실전 실험

### 실험 1: TCP_NODELAY 효과 측정

```python
# nodelay_test.py
import socket, time

def measure_latency(host, port, nodelay=True, iterations=100):
    s = socket.socket()
    s.connect((host, port))
    if nodelay:
        s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
    
    latencies = []
    for _ in range(iterations):
        start = time.perf_counter()
        s.send(b"PING\r\n")
        s.recv(1024)
        latencies.append((time.perf_counter() - start) * 1000)
    
    s.close()
    avg = sum(latencies) / len(latencies)
    p99 = sorted(latencies)[int(len(latencies) * 0.99)]
    print(f"TCP_NODELAY={'ON' if nodelay else 'OFF'}: avg={avg:.2f}ms p99={p99:.2f}ms")

# Redis 서버 필요
measure_latency('localhost', 6379, nodelay=True)
measure_latency('localhost', 6379, nodelay=False)
```

```bash
$ redis-server &
$ python3 nodelay_test.py
TCP_NODELAY=ON:  avg=0.21ms p99=0.45ms
TCP_NODELAY=OFF: avg=0.22ms p99=203.45ms  ← p99에서 200ms 지연!
# Nagle 알고리즘 + Delayed ACK 조합의 실제 영향
```

### 실험 2: SO_KEEPALIVE 동작 확인

```bash
# keepalive 옵션 설정 후 소켓 상태 확인
# Python으로 keepalive 소켓 생성
$ python3 -c "
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 5)    # 5초
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 2)   # 2초 간격
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 3)     # 3회
s.connect(('httpbin.org', 80))
print(f'연결됨. PID={__import__(\"os\").getpid()}')
time.sleep(300)
" &
PID=$!

# keepalive timer 확인
$ ss -toni | grep "keepalive"
  timer:(keepalive,4sec,0)  ← 4초 후 첫 probe 예정

# tcpdump로 keepalive probe 패킷 관찰
$ tcpdump -i eth0 "tcp and host httpbin.org" -n
# 5초 후: keepalive probe (data 없는 ACK)
# 응답 있으면: keepalive ack
```

### 실험 3: SO_REUSEPORT 분산 확인

```bash
# SO_REUSEPORT로 두 프로세스가 같은 포트 공유
# 프로세스 1
$ python3 -c "
import socket, os
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
s.bind(('', 8888))
s.listen(100)
print(f'Worker A (PID={os.getpid()}) 대기 중')
while True:
    conn, addr = s.accept()
    print(f'Worker A: {addr} 수신')
    conn.close()
" &

# 프로세스 2
$ python3 -c "
import socket, os
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
s.bind(('', 8888))
s.listen(100)
print(f'Worker B (PID={os.getpid()}) 대기 중')
while True:
    conn, addr = s.accept()
    print(f'Worker B: {addr} 수신')
    conn.close()
" &

# 10개 동시 연결
$ for i in $(seq 1 10); do nc -q 0 localhost 8888 & done
# Worker A와 Worker B가 번갈아 연결 처리 확인
```

---

## 📊 성능/비용 비교

| 소켓 옵션 | 효과 | 비용 | 적합한 경우 |
|---------|-----|-----|-----------|
| TCP_NODELAY | 레이턴시 감소 | 패킷 수 증가 | Redis, gRPC, 게임 서버 |
| SO_KEEPALIVE | 죽은 연결 감지 | 주기적 probe 패킷 | 장기 연결 풀, DB 연결 |
| SO_REUSEPORT | Accept 경합 제거 | 소켓 수 증가 | Nginx, 멀티 워커 서버 |
| SO_LINGER(l=0) | TIME_WAIT 제거 | 데이터 유실 위험 | 절대 프로덕션 사용 금지 |

**Nagle 알고리즘 비교**:

```
요청/응답 크기    TCP_NODELAY=OFF    TCP_NODELAY=ON
10 bytes req    최대 200ms 지연      < 1ms
1 KB req        즉시 전송             즉시 전송 (MSS 이내)
10 KB req       즉시 전송             즉시 전송

결론:
  소형 패킷(< MSS): TCP_NODELAY=ON 필수
  대형 패킷(> MSS): TCP_NODELAY 무관
```

---

## ⚖️ 트레이드오프

**TCP_NODELAY: 레이턴시 vs 패킷 효율**

TCP_NODELAY를 켜면 모든 쓰기가 즉시 패킷으로 나간다. Redis처럼 수십 바이트의 명령어를 빠르게 주고받는 경우에는 필수적이다. 하지만 HTTP 응답처럼 애플리케이션이 여러 번의 `write()`로 하나의 응답을 구성하는 경우, TCP_NODELAY는 여러 작은 패킷을 만들어 오히려 비효율적이다. `TCP_CORK`로 임시 버퍼링 후 한 번에 보내는 방법도 있다.

**SO_KEEPALIVE: OS 설정 vs 애플리케이션 레벨 헬스체크**

OS의 `tcp_keepalive_time` 기본값(2시간)은 대부분의 방화벽 idle timeout(30~60초)보다 훨씬 길다. 방화벽이 연결을 끊기 전에 keepalive가 동작하려면 OS 파라미터를 낮춰야 한다. 또는 JDBC, Redis 클라이언트처럼 애플리케이션 레벨의 connection validation(소켓 레벨 핑 전송)으로 동일한 효과를 낼 수 있다.

---

## 📌 핵심 정리

```
TCP_NODELAY:
  Nagle 알고리즘 비활성화 → 즉시 전송
  Redis, gRPC 등 레이턴시 민감한 서비스 필수
  setsockopt(IPPROTO_TCP, TCP_NODELAY, &1, ...)

SO_KEEPALIVE:
  커널이 주기적 probe로 죽은 연결 감지
  OS 파라미터: tcp_keepalive_time (기본 7200초) 낮춰야 실용적
  방화벽 idle timeout보다 짧게 설정 필수
  setsockopt(SOL_SOCKET, SO_KEEPALIVE, &1, ...)

SO_REUSEPORT:
  여러 프로세스/스레드가 동일 포트를 독립 소켓으로 listen
  Accept 경합 제거, 처리량 향상
  Nginx reuseport, Tomcat NIO에서 활용
  setsockopt(SOL_SOCKET, SO_REUSEPORT, &1, ...)

SO_LINGER(l=0):
  RST로 즉시 종료, TIME_WAIT 없앰
  프로덕션 사용 금지 (데이터 유실 위험)

진단 명령어:
  ss -toni         → 소켓 옵션 상태 (nonagle, keepalive timer)
  tcpdump          → keepalive probe 패킷 직접 관찰
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Boot의 `RestTemplate`으로 외부 API를 호출할 때 `TCP_NODELAY`가 중요한가?

<details>
<summary>해설 보기</summary>

일반적으로 중요하지 않다. HTTP 요청은 헤더 + 바디를 한 번에 보내거나, 애플리케이션이 `OutputStream`에 모아서 전송하기 때문에 패킷 크기가 충분히 크다. Nagle 알고리즘의 버퍼링 효과가 발생하지 않는다. 하지만 gRPC처럼 소형 메시지를 빈번히 주고받는 경우에는 `TCP_NODELAY`가 중요하다. `RestTemplate`의 기반인 `HttpURLConnection`은 기본적으로 `TCP_NODELAY`를 설정하지 않는다.

</details>

**Q2.** DB 연결 풀에서 10분 동안 사용하지 않은 연결을 재사용할 때 오류가 발생한다. `SO_KEEPALIVE`를 설정했는데도 왜 그런가?

<details>
<summary>해설 보기</summary>

`tcp_keepalive_time` 기본값(7200초 = 2시간)이 10분보다 훨씬 크기 때문이다. 10분 동안 연결이 유휴 상태이면 방화벽이 해당 NAT 항목을 삭제할 수 있다(대부분의 방화벽은 5~30분 idle timeout). 이후 keepalive probe가 전송되도 방화벽이 RST를 보내거나 단순히 드롭한다. 해결책은 `tcp_keepalive_time=300`(5분)으로 낮추거나, JDBC 드라이버의 `validationQuery`를 설정해 연결 사용 전 유효성을 확인하는 것이다.

</details>

**Q3.** Nginx가 `reuseport`를 사용할 때 Rolling Restart 중 연결 손실이 발생하는 이유와 해결 방법은?

<details>
<summary>해설 보기</summary>

기존 worker가 종료되면 해당 worker의 소켓도 닫힌다. 그 소켓의 Accept Queue에 대기 중이던 연결들은 RST를 받아 연결이 끊긴다. 반면 `SO_REUSEPORT` 없는 전통적 방식은 마스터 프로세스가 소켓을 유지하므로 연결이 손실되지 않는다. 해결책으로는 rolling restart 전에 해당 worker에 새 연결이 들어오지 않도록 로드 밸런서에서 먼저 제거하고, 기존 연결 처리가 완료된 후 종료하는 graceful shutdown을 구현한다. Nginx의 `worker_shutdown_timeout` 설정이 이를 지원한다.

</details>

---

**[⬅️ 이전: TCP 소켓 상태](./02-tcp-socket-state.md)** | **[홈으로 🏠](../README.md)** | **[다음: 커널 네트워크 파라미터 튜닝 ➡️](./04-kernel-network-tuning.md)**
