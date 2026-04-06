# 02. TCP 소켓 상태와 커널 — Accept Queue의 한계

## 🎯 핵심 질문

- SYN Backlog와 Accept Queue는 어떻게 다르고, 각각 무엇이 가득 차면 어떤 일이 생기는가?
- `net.core.somaxconn`과 `tcp_max_syn_backlog`는 어느 큐를 제어하는가?
- 클라이언트가 `Connection refused`와 무응답(timeout)을 받는 상황은 어떻게 다른가?
- Kubernetes 환경에서 Pod 시작 직후 연결 오류가 자주 발생하는 커널 레벨 원인은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**배포 직후 일부 요청이 `Connection refused`로 실패한다.** Spring Boot가 완전히 시작되기 전에 Kubernetes가 readiness probe를 통과시키거나, 새 Pod가 Listen 소켓을 열기 전에 트래픽이 들어오면 발생한다. 더 정확히는 `listen(backlog)` 호출 전이거나, Accept Queue가 짧은 순간 가득 찬 것이다.

**트래픽 급증 시 일부 요청이 타임아웃된다.** `Connection refused`가 아닌 무응답이라면 SYN Backlog가 가득 찬 것이다. 커널이 SYN 패킷을 조용히 드롭하면 클라이언트는 재전송을 시도하다가 타임아웃된다. `netstat -s | grep "SYNs to LISTEN"` 카운터가 올라가면 이 상황이다.

**Nginx 앞에 Node.js 서버가 있다. Nginx 측에서 간헐적 `Connection reset by peer`가 발생한다.** Node.js의 Accept Queue가 가득 차서 새 연결을 받자마자 RST를 보내는 경우다. `net.core.somaxconn`을 늘리고 Node.js의 `server.listen()` backlog 인자도 함께 늘려야 한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: somaxconn만 늘리고 애플리케이션 backlog를 그대로 둠
$ sysctl -w net.core.somaxconn=4096
# 그런데 Java 코드:
ServerSocket serverSocket = new ServerSocket(8080, 50);  # backlog=50
# → min(4096, 50) = 50 → Accept Queue 여전히 50개 제한!

# 실수 2: SYN Backlog와 Accept Queue 구분 없이 설정
$ sysctl -w net.core.somaxconn=1024  # Accept Queue 제어
# SYN Backlog는 tcp_max_syn_backlog가 제어
$ sysctl net.ipv4.tcp_max_syn_backlog  # 이걸 설정 안 함

# 실수 3: SYN flood 공격인지 정상 부하인지 구분 못 함
$ netstat -s | grep "SYNs to LISTEN"
    1234 SYNs to LISTEN sockets dropped
# "공격인가?" → 아닐 수도 있음. 단순 SYN Backlog 포화일 수 있음
# tcp_syncookies=1이 설정돼 있으면 SYN flood도 어느 정도 방어
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 현재 Listen 소켓 큐 상태 확인
$ ss -ltnp
State    Recv-Q  Send-Q  Local Address:Port
LISTEN   0       128     0.0.0.0:80       ← Accept Queue 크기 128 (Send-Q)
LISTEN   5       128     0.0.0.0:8080     ← Recv-Q=5: 대기 중인 연결 5개

# LISTEN 상태에서:
# Recv-Q = Accept Queue에 있는 연결 수 (accept() 대기 중)
# Send-Q = Accept Queue 최대 크기 (listen backlog)

# Accept Queue 포화 확인
$ netstat -s | grep -i "listen\|overflow"
    1234 times the listen queue of a socket overflowed
    5678 SYNs to LISTEN sockets dropped

# 최적 설정
$ sysctl -w net.core.somaxconn=4096
$ sysctl -w net.ipv4.tcp_max_syn_backlog=8192

# Java 서버에서 backlog 설정
ServerSocket ss = new ServerSocket(8080, 4096);  # backlog 명시
# 또는 Spring Boot:
# server.tomcat.accept-count=4096
```

---

## 🔬 내부 동작 원리

### TCP 3-way Handshake와 두 개의 큐

```
클라이언트                  커널 (서버)                 애플리케이션
    │                          │                            │
    │──── SYN ──────────────→  │                            │
    │                 [SYN Backlog에 추가]                    │
    │                 half-open connection                  │
    │                          │                            │
    │ ←─── SYN+ACK ──────────  │                            │
    │                          │                            │
    │──── ACK ──────────────→  │                            │
    │                 [SYN Backlog에서 제거]                  │
    │                 [Accept Queue에 추가]                   │
    │                 ESTABLISHED 상태                       │
    │                          │                            │
    │                          │        accept() 호출 ──────→│
    │                          │ ←── 연결 반환 (fd) ───────   │
    │                 [Accept Queue에서 제거]                 │
    │                          │                            │

SYN Backlog (Incomplete Queue):
  - SYN 받은 후 ACK 기다리는 반완성 연결 저장
  - 크기: net.ipv4.tcp_max_syn_backlog

Accept Queue (Complete Queue):
  - 3-way handshake 완료된 연결 저장
  - 크기: min(somaxconn, listen() backlog)
```

### 큐가 가득 찼을 때 동작

```
Accept Queue 포화 시:
  새로운 3-way handshake 완료 → Accept Queue 이미 가득 참
  커널 동작: RST 전송 또는 SYN+ACK 재전송 중단

  tcp_abort_on_overflow = 0 (기본):
    마지막 ACK를 무시 → 클라이언트에게 RST 미전송
    → 클라이언트: SYN+ACK를 재전송 (잠시 후 재시도)
    → 애플리케이션이 accept()를 빠르게 처리하면 자연 회복

  tcp_abort_on_overflow = 1:
    RST 전송 → 클라이언트: "Connection refused"
    → 즉각적 실패 (빠른 실패 유도)

SYN Backlog 포화 시:
  tcp_syncookies = 0:
    새 SYN 드롭 (무응답)
    → 클라이언트: 재전송 후 결국 타임아웃

  tcp_syncookies = 1 (기본):
    SYN cookie 방식으로 backlog 없이 처리
    → SYN Backlog 포화에도 연결 수립 가능
    → SYN flood 공격 방어 효과
```

### TCP 연결 상태 전이

```
클라이언트 측:
  CLOSED → SYN_SENT → ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2
                                   → TIME_WAIT → CLOSED

서버 측:
  CLOSED → LISTEN → SYN_RCVD → ESTABLISHED → CLOSE_WAIT
                                             → LAST_ACK → CLOSED

중요 상태:
  TIME_WAIT: 활성 종료 측 (FIN 먼저 보낸 쪽)
    - 지속 시간: 2 * MSL (Maximum Segment Lifetime, 보통 60초)
    - 목적: 지연된 패킷이 새 연결로 혼동되지 않도록
    - 다량 발생: 단기 연결이 많은 서버 (REST API, HTTP/1.0)

  CLOSE_WAIT: 수동 종료 측 (FIN을 받았지만 아직 close() 안 함)
    - 애플리케이션이 close() 호출 안 하면 CLOSE_WAIT에 고착
    - 다량 발생: fd 누수 또는 종료 처리 누락 버그

  FIN_WAIT_2: 반만 닫힌 상태 (FIN 보냈지만 상대방 FIN 아직 안 옴)
    - 상대방이 close()를 오래 안 하면 이 상태 지속
    - tcp_fin_timeout으로 타임아웃 설정 가능
```

### Kubernetes Pod 시작 시 연결 오류 원인

```
문제 상황:
  새 Pod 배포 → Service에 Endpoint 등록 → 트래픽 유입
  → Pod가 아직 준비 안 됨 → 연결 실패

커널 레벨 원인:
  1. Listen 소켓 미개방: 포트 아직 bind/listen 안 됨
     → 커널이 RST 전송 → "Connection refused"

  2. Accept Queue 포화: 급격한 트래픽 유입
     → 큐가 순간 넘침 → 일부 연결 드롭/RST

  3. Application 초기화 지연: listen()은 됐지만 accept() 못 함
     → Accept Queue가 가득 참 → 새 연결 거부

해결:
  readinessProbe 정확한 설정:
    httpGet: /actuator/health/readiness
    initialDelaySeconds: 10  ← 충분한 시작 대기
    failureThreshold: 3

  preStop 훅:
    exec:
      command: ["sleep", "5"]  ← 연결 종료 후 트래픽 차단

  backlog 충분히 설정:
    server.tomcat.accept-count=1024  ← 급증 트래픽 버퍼
```

---

## 💻 실전 실험

### 실험 1: Accept Queue 포화 재현

```python
# accept_queue_test.py
import socket, time, threading

def slow_server(backlog=5):
    """backlog=5로 Accept Queue를 작게 설정"""
    s = socket.socket()
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(('', 9998))
    s.listen(backlog)  # Accept Queue = min(somaxconn, 5)
    print(f"서버 시작 (backlog={backlog})")
    time.sleep(5)  # accept()를 5초 지연 → Accept Queue 포화 유도
    while True:
        conn, _ = s.accept()
        print(f"연결 처리: {conn.getpeername()}")
        conn.close()

def many_clients():
    """동시에 10개 연결 시도"""
    time.sleep(0.5)
    for i in range(10):
        def connect(n):
            try:
                c = socket.socket()
                c.settimeout(3)
                c.connect(('localhost', 9998))
                print(f"클라이언트 {n}: 성공")
                c.close()
            except Exception as e:
                print(f"클라이언트 {n}: 실패 - {e}")
        threading.Thread(target=connect, args=(i,)).start()

threading.Thread(target=slow_server).start()
threading.Thread(target=many_clients).start()
```

```bash
$ python3 accept_queue_test.py
# backlog=5이므로 5개는 Accept Queue에서 대기
# 나머지는 거부 또는 무응답

# 별도 터미널에서 Accept Queue 상태 확인
$ ss -ltnp | grep 9998
LISTEN 5 5 0.0.0.0:9998  ← Recv-Q=5 (대기 중인 연결 수)

# 오버플로 카운터 확인
$ netstat -s | grep overflow
    5 times the listen queue of a socket overflowed
```

### 실험 2: TCP 상태 분포 모니터링

```bash
# 현재 TCP 상태 분포
$ ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
   300 ESTAB
    40 TIME_WAIT
    10 CLOSE_WAIT   ← 이게 증가하면 버그!
     5 SYN_RECV
     2 LISTEN

# TIME_WAIT 대량 발생 원인 확인 (단기 연결 패턴)
$ ss -tan state time-wait | wc -l
40

# CLOSE_WAIT 고착 소켓 상세 (fd 누수 의심)
$ ss -tanp state close-wait
# → 어느 프로세스의 어느 fd인지 확인

# 1초 간격으로 상태 변화 모니터링
$ watch -n 1 "ss -s"
```

### 실험 3: SYN Backlog 포화 진단

```bash
# SYN Backlog 관련 카운터
$ netstat -s | grep -E "SYN|backlog|overflow"
    1234 SYNs to LISTEN sockets dropped    ← SYN Backlog 포화
    5678 times the listen queue of a socket overflowed  ← Accept Queue 포화

# SYN cookie 동작 확인
$ sysctl net.ipv4.tcp_syncookies
net.ipv4.tcp_syncookies = 1  ← 활성화됨 (SYN flood 방어)

$ netstat -s | grep cookie
    0 SYN cookies sent     ← 0이면 SYN Backlog 포화 없음
    5 SYN cookies received ← SYN cookie 방어 발동됨
```

---

## 📊 성능/비용 비교

| 큐 | 제어 파라미터 | 포화 시 동작 | 클라이언트 경험 |
|---|------------|------------|--------------|
| SYN Backlog | `tcp_max_syn_backlog` | SYN 드롭 | 타임아웃 (재전송 후) |
| Accept Queue | `somaxconn` + `listen(backlog)` | RST 또는 드롭 | Connection refused 또는 타임아웃 |

**backlog 크기별 트래픽 버퍼링 능력**:

```
backlog = 128 (Nginx 기본):
  accept() 지연 허용: 128개 연결 * 평균 처리 시간
  10ms 처리 시간: 128 * 10ms = 1.28초 트래픽 버퍼링
  초당 1000 RPS: 1.28초 × 1000 = 1280 연결 처리

backlog = 4096:
  동일 조건에서 40.96초 트래픽 버퍼링 가능
  급격한 트래픽 급증 처리에 유리
  메모리: 연결당 수 KB → 4096개 = 수십 MB (감당 가능)
```

---

## ⚖️ 트레이드오프

**큰 backlog vs 빠른 실패**

backlog를 크게 설정하면 트래픽 급증 시 연결을 더 많이 수용할 수 있다. 하지만 서버가 실제로 처리 불가능한 상황에서도 연결을 받아두어 클라이언트가 오래 기다리게 된다. `tcp_abort_on_overflow=1`로 큐 포화 시 즉시 RST를 보내면 클라이언트가 빠르게 다른 서버로 재시도할 수 있다.

**TIME_WAIT의 필요성과 비용**

TIME_WAIT는 지연 패킷으로 인한 연결 혼동을 막는 중요한 메커니즘이다. 하지만 단기 연결이 많은 서버(API 서버, 마이크로서비스)에서 TIME_WAIT가 수만 개 쌓이면 포트 고갈로 새 연결이 안 되는 상황이 발생한다. `tcp_tw_reuse=1`로 완화하거나, HTTP Keep-Alive로 연결을 재사용해 근본 원인을 줄이는 것이 낫다.

---

## 📌 핵심 정리

```
두 개의 큐:
  SYN Backlog: 3-way handshake 진행 중인 연결
    → tcp_max_syn_backlog로 크기 제어
    → 포화 시 SYN 드롭 → 클라이언트 타임아웃
  Accept Queue: 완성된 연결 (accept() 대기)
    → min(somaxconn, listen backlog)
    → 포화 시 RST 또는 드롭 → Connection refused 또는 타임아웃

ss LISTEN 상태:
  Recv-Q = 현재 대기 중인 연결 수
  Send-Q = Accept Queue 최대 크기

TCP 상태 진단:
  TIME_WAIT 다량: 단기 연결 패턴 (Keep-Alive 활성화 권장)
  CLOSE_WAIT 다량: fd 누수 또는 close() 미호출 버그

핵심 설정:
  net.core.somaxconn = 4096       (Accept Queue 상한)
  net.ipv4.tcp_max_syn_backlog = 8192  (SYN Backlog)
  net.ipv4.tcp_syncookies = 1     (SYN flood 방어)

핵심 진단:
  ss -ltnp              → LISTEN 소켓 큐 상태
  netstat -s | grep overflow → 큐 오버플로 카운터
  ss -s                 → 전체 TCP 상태 요약
```

---

## 🤔 생각해볼 문제

**Q1.** Nginx를 `worker_processes 4`로 설정했다. 4개의 worker 프로세스가 모두 같은 포트 80을 `listen()`한다. 같은 포트를 여러 프로세스가 listen할 수 있는 이유는?

<details>
<summary>해설 보기</summary>

`SO_REUSEPORT` 소켓 옵션 때문이다. 이 옵션 없이는 하나의 포트에 하나의 소켓만 `listen()`할 수 있다. `SO_REUSEPORT`를 설정하면 여러 프로세스가 같은 포트를 각자 `listen()`할 수 있고, 커널이 Accept Queue를 각 소켓에 분산한다. 각 worker가 독립적인 Accept Queue를 가지므로 `accept()` 경합이 없어 처리량이 향상된다. Nginx가 `SO_REUSEPORT`를 사용하는 이유다. 다음 문서에서 더 자세히 다룬다.

</details>

**Q2.** 클라이언트가 서버에 연결 시 `Connection refused`를 받았다. 서버 프로세스는 실행 중이다. 어떤 원인들이 가능한가?

<details>
<summary>해설 보기</summary>

세 가지 주요 원인이 있다. 첫째, 프로세스가 해당 포트를 아직 `listen()` 상태로 만들지 않은 경우(시작 중이거나 다른 포트). 둘째, 방화벽이 RST를 보내는 경우. 셋째, Accept Queue가 포화되어 `tcp_abort_on_overflow=1` 설정으로 RST를 전송하는 경우다. `ss -ltnp | grep <port>`로 포트가 LISTEN 상태인지 확인하고, `netstat -s | grep overflow`로 큐 포화를 확인한다.

</details>

**Q3.** 서버에 `TIME_WAIT` 소켓이 4만 개 쌓였다. 이것이 새 연결 수립을 방해할 수 있는가?

<details>
<summary>해설 보기</summary>

서버 측에서는 `TIME_WAIT` 소켓이 많아도 새 연결 수립에 직접적인 영향이 없다. 서버는 항상 같은 포트(80, 8080 등)를 LISTEN하고, 클라이언트마다 다른 (src_ip, src_port) 조합을 가지므로 소켓이 겹치지 않는다. 문제가 되는 것은 클라이언트 측이다. 클라이언트가 서버에 새 연결을 만들 때 source port를 소비하는데, `TIME_WAIT` 상태의 연결과 동일한 4-tuple(src_ip, src_port, dst_ip, dst_port)을 사용할 수 없다. 클라이언트가 같은 서버로 많은 단기 연결을 만들면 source port(64511개 범위)가 고갈될 수 있다. `tcp_tw_reuse=1`로 완화하거나 연결 재사용(Keep-Alive)으로 해결한다.

</details>

---

**[⬅️ 이전: 소켓과 커널 버퍼](./01-socket-kernel-buffer.md)** | **[홈으로 🏠](../README.md)** | **[다음: 소켓 옵션 ➡️](./03-socket-options.md)**
