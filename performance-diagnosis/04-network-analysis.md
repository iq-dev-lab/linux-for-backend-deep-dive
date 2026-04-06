# 04. 네트워크 분석 — 소켓 상태와 TIME_WAIT 진단

## 🎯 핵심 질문

- `ss -tanp`로 `ESTABLISHED`, `TIME_WAIT`, `CLOSE_WAIT` 각 상태를 어떻게 해석하는가?
- TIME_WAIT가 수만 개 쌓이는 원인과 정확한 해결 방법은?
- `sar -n DEV 1`으로 네트워크 처리량이 한계에 가까워지는 것을 어떻게 감지하는가?
- `Recv-Q` 누적이 의미하는 것과 근본 원인을 어떻게 찾는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**API 서버에서 "Cannot assign requested address" 오류가 발생한다.** 클라이언트 측 포트가 고갈된 것이다. `ss -s`에서 `timewait` 수가 포트 범위를 초과하면 새 연결을 만들 수 없다. 마이크로서비스 체인에서 HTTP Keep-Alive 없이 짧은 연결을 반복하면 발생한다.

**`CLOSE_WAIT` 상태 소켓이 수백 개 쌓여 있다.** 상대방이 FIN을 보냈는데 애플리케이션이 소켓을 닫지 않은 것이다. 코드에서 `close()` 또는 `socket.close()`를 누락한 버그다. 시간이 지나도 줄어들지 않으면 fd 누수다.

**Nginx 접속이 가끔 느린데 CPU, 디스크 모두 정상이다.** 네트워크 처리량이 NIC 한계에 가까워지거나 수신 큐가 포화됐을 가능성이다. `sar -n DEV 1`으로 처리량을, `ss -tmn`으로 `Recv-Q` 상태를 확인해야 한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: TIME_WAIT를 에러로 오해
$ ss -tan | grep TIME_WAIT | wc -l
2000

# "TIME_WAIT가 2000개나! 뭔가 잘못됐다" → 오해
# TIME_WAIT는 정상적인 TCP 종료 과정
# 문제는 포트 고갈이 발생할 때만

# 실수 2: CLOSE_WAIT를 TIME_WAIT와 혼동
# TIME_WAIT = 활성 종료 측 (FIN 먼저 보낸 쪽)
# CLOSE_WAIT = 수동 종료 측에서 아직 close() 안 함
# → CLOSE_WAIT는 애플리케이션 버그 신호!

# 실수 3: tcp_tw_recycle 사용 시도 (절대 금지)
$ sysctl -w net.ipv4.tcp_tw_recycle=1
# → Linux 4.12에서 완전 삭제됨
# → 구형 커널에서는 NAT 환경에서 연결 오류 유발
# → 올바른 방법: tcp_tw_reuse=1 (클라이언트만)
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 네트워크 진단 단계별 접근

# Step 1: 소켓 상태 분포 확인
$ ss -s
TCP:   5234 (estab 500, closed 200, orphaned 10, timewait 4500)

# Step 2: 상태별 상세 확인
$ ss -tan state time-wait | wc -l      # TIME_WAIT 수
$ ss -tan state close-wait | wc -l     # CLOSE_WAIT 수 (버그 신호!)
$ ss -tan state established | wc -l    # 활성 연결 수

# Step 3: Recv-Q 높은 소켓 확인
$ ss -tmn | awk 'NR>1 && $2 > 1000 {print "Recv-Q 높음:", $0}'

# Step 4: 네트워크 처리량 확인
$ sar -n DEV 1 5
# rxpck/s, txpck/s, rxkB/s, txkB/s

# Step 5: TIME_WAIT 대상 서버 확인
$ ss -tan state time-wait | awk '{print $5}' | \
  cut -d: -f1 | sort | uniq -c | sort -rn | head
# 어느 서버로의 TIME_WAIT가 많은지
```

---

## 🔬 내부 동작 원리

### TCP 상태 완전 진단 가이드

```
중요 TCP 상태와 진단 의미:

ESTABLISHED: 정상 활성 연결
  수가 많으면: 트래픽 많음 (정상)
  수가 갑자기 줄면: 연결 끊김 이벤트 발생

SYN_RECV: 3-way handshake 진행 중 (SYN Backlog)
  수가 많고 지속적이면: SYN flood 공격 또는 높은 연결 요청
  → tcp_syncookies=1 확인

TIME_WAIT: 활성 종료 측 (FIN 먼저 보낸 쪽)
  정상적 범주: 수백~수천 개
  문제 범주: 포트 범위 초과 시
  → 클라이언트: tcp_tw_reuse=1, Keep-Alive 활성화
  → 서버: 영향 제한적 (서버는 항상 같은 포트 listen)

CLOSE_WAIT: 수동 종료 측 (FIN 받았지만 close() 안 함)
  ★ 0이어야 정상!
  수가 쌓이면: 애플리케이션 버그 (fd 누수)
  → close() 누락, try-with-resources 미사용

FIN_WAIT_2: FIN 보냈지만 상대방 FIN 아직 안 옴
  지속되면: 상대방이 close() 호출 안 함
  → tcp_fin_timeout으로 타임아웃 설정

LAST_ACK: FIN 보내고 마지막 ACK 대기
  지속되면: 네트워크 패킷 손실
```

### TIME_WAIT 포트 고갈 메커니즘

```
마이크로서비스 A → B 호출 (Keep-Alive 없음):

각 요청마다:
  1. A가 새 소켓 생성 (임시 포트 할당, 예: 54321)
  2. A가 B:8080에 connect()
  3. 요청/응답 완료
  4. A가 FIN 전송 (A가 먼저 종료)
  5. A의 소켓: TIME_WAIT 상태 (2 * MSL = 60초)
  6. 60초 동안 포트 54321은 재사용 불가

포트 범위: net.ipv4.ip_local_port_range = 32768~60999
사용 가능 포트: 28231개

초당 1000 req → TIME_WAIT 1000개/초 생성
60초 동안 유지 → 최대 60,000개 동시 TIME_WAIT
28231 < 60,000 → 포트 고갈!

"Cannot assign requested address" 오류 발생:
  A가 새 소켓을 만들려 하지만 사용 가능한 임시 포트가 없음

해결 방법 (우선순위 순):
  1. HTTP Keep-Alive 활성화 (연결 재사용 → 새 TIME_WAIT 없음)
  2. Connection Pool 사용
  3. tcp_tw_reuse=1 (타임스탬프 기반 재사용, 클라이언트)
  4. ip_local_port_range 확장 (1024~65535)
  5. tcp_fin_timeout 단축 (30초)
```

### CLOSE_WAIT 버그 찾기

```bash
# CLOSE_WAIT 소켓 상세 확인
$ ss -tanp state close-wait
State    Recv-Q  Send-Q  Local Addr:Port  Peer Addr:Port
CLOSE-WAIT  0      0    10.0.0.1:8080    10.0.0.2:45678   users:(("java",pid=1234,fd=45))
CLOSE-WAIT  0      0    10.0.0.1:8080    10.0.0.3:45679   users:(("java",pid=1234,fd=46))

# → java 프로세스 PID=1234 의 fd 45, 46이 CLOSE_WAIT 상태

# 해당 fd가 어떤 파일/소켓인지
$ ls -la /proc/1234/fd/45
lrwx------ fd/45 -> socket:[123456]

# Java에서 어떤 코드가 이 소켓을 열었는지
$ jstack 1234 | grep -A 20 "close_wait\|socket"

# 시간에 따른 CLOSE_WAIT 수 추이
$ while true; do
    count=$(ss -tan state close-wait | wc -l)
    echo "$(date +%H:%M:%S) CLOSE_WAIT=$count"
    sleep 5
  done
# 계속 증가 = fd 누수 버그
# 일정 수준 유지 = 완료 처리 속도 ≤ 새로 생기는 속도
```

### sar로 네트워크 처리량 모니터링

```bash
$ sar -n DEV 1 10

11:23:45   IFACE   rxpck/s  txpck/s  rxkB/s  txkB/s  rxcmp/s txcmp/s rxmcst/s
11:23:46   lo        100.0    100.0    50.0    50.0       0.0     0.0      0.0
11:23:46   eth0    95000.0  85000.0  7500.0  8500.0       0.0     0.0      0.0

# rxkB/s: 초당 수신 KB
# txkB/s: 초당 송신 KB

# 1Gbps NIC 최대: 125MB/s = 128000 kB/s
# eth0: rxkB/s=7500 (→ 7.5MB/s, 용량의 6%)  → 여유 있음
# eth0: rxpck/s=95000 → 95K 패킷/초 (소형 패킷 집약적)

# NIC 한계 접근 확인
$ cat /proc/net/dev | grep eth0
eth0: rxbytes rxpackets rxerrs rxdrop rxfifo rxframe rxcompressed rxmulticast
      txbytes txpackets txerrs txdrop txfifo txcolls txcarrier txcompressed

# rxdrop 증가 = NIC 수신 버퍼 포화 (패킷 드롭)
# rxerrs 증가 = 네트워크 오류

# 실시간 드롭 확인
$ watch -n 1 'cat /proc/net/dev | grep eth0 | awk "{print \"drop:\", \$5}"'
```

---

## 💻 실전 실험

### 실험 1: TIME_WAIT 생성 및 포트 고갈 재현

```bash
# Keep-Alive 없는 단기 연결 반복
$ cat > /tmp/short_conn.py << 'EOF'
import socket, time, sys
target = ('localhost', 80)  # 또는 실제 서버
while True:
    s = socket.socket()
    s.connect(target)
    s.send(b"GET / HTTP/1.0\r\nHost: localhost\r\n\r\n")
    s.recv(1024)
    s.close()  # → TIME_WAIT 생성
    time.sleep(0.001)
EOF
$ python3 /tmp/short_conn.py &

# TIME_WAIT 증가 관찰
$ watch -n 1 "ss -s | grep timewait"
```

### 실험 2: CLOSE_WAIT 재현 (fd 누수 시뮬레이션)

```python
# close_wait_demo.py
# 서버: 연결을 받지만 close() 안 함
import socket, threading

def server():
    s = socket.socket()
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(('', 9996))
    s.listen(10)
    while True:
        conn, _ = s.accept()
        # conn.close()를 호출하지 않음! → 클라이언트 종료 시 CLOSE_WAIT

threading.Thread(target=server, daemon=True).start()

# 클라이언트가 먼저 종료
import time
clients = []
for i in range(5):
    c = socket.socket()
    c.connect(('localhost', 9996))
    clients.append(c)

time.sleep(1)
for c in clients:
    c.close()  # 클라이언트 종료 → 서버 측 CLOSE_WAIT

time.sleep(5)
print(f"CLOSE_WAIT 확인: ss -tan state close-wait")
input("Press Enter...")
```

```bash
$ python3 close_wait_demo.py &
$ ss -tan state close-wait
# CLOSE_WAIT 소켓 확인
```

### 실험 3: 네트워크 처리량 측정 및 한계 확인

```bash
# iperf3로 네트워크 처리량 측정
$ iperf3 -s &  # 서버
$ iperf3 -c localhost -t 30 -P 4  # 4 병렬 스트림

# 동시에 sar으로 모니터링
$ sar -n DEV 1 30 | grep eth0

# txkB/s가 NIC 한계(125000 kB/s = 1Gbps)에 가까워지면:
# → 패킷 드롭 발생 가능
# /proc/net/dev의 txdrop 확인

# 실제 서버 트래픽 패턴 확인
$ sar -n DEV 1 5
# 패킷 크기 계산: txkB/s / txpck/s = 평균 패킷 크기
# 64B 이하 소형 패킷 집약 → 패킷 처리율 한계
# 1460B+ 대형 패킷 → 처리량 한계
```

---

## 📊 성능/비용 비교

**소켓 상태별 의미 요약**:

```
상태          정상 범위      주의 기준        원인과 해결
ESTABLISHED   부하 비례      갑자기 급감      연결 끊김 이벤트
TIME_WAIT     수백~수천      포트 범위 초과   Keep-Alive, tcp_tw_reuse
CLOSE_WAIT    0이 정상       0 초과          fd 누수 버그 (close() 미호출)
SYN_RECV      순간적 발생   지속 수백+       SYN flood, backlog 부족
FIN_WAIT_2   순간적 발생    지속 수백+       상대방 close() 지연
```

**네트워크 처리량 한계 지표**:

```
1Gbps NIC 기준:
  최대 처리량: 125 MB/s = 128,000 kB/s
  최대 패킷 수: 1,488,095 pps (64바이트 패킷)

병목 지점:
  처리량 한계: txkB/s + rxkB/s ≈ 128,000 kB/s
  패킷 한계:   txpck/s + rxpck/s > 수십만 pps
  CPU 한계:    hi/si가 높아지며 패킷 드롭
```

---

## ⚖️ 트레이드오프

**TIME_WAIT를 제거하려는 유혹**

`SO_LINGER(l=0)`으로 RST를 보내 TIME_WAIT를 없애려는 시도가 있다. 데이터 유실 위험이 있고 상대방에서 `ECONNRESET` 오류가 발생한다. 프로덕션에서 절대 사용하면 안 된다. `tcp_tw_reuse=1`은 타임스탬프 기반으로 안전하게 재사용하는 올바른 방법이다.

**Recv-Q 누적과 애플리케이션 처리 속도**

`Recv-Q > 0`은 애플리케이션이 커널 수신 버퍼에서 데이터를 읽는 속도가 데이터가 들어오는 속도보다 느린 것이다. 일시적이면 문제없지만, 지속되면 수신 버퍼가 포화되어 TCP Zero Window가 발생하고 송신측이 멈춘다. 애플리케이션 처리 속도를 높이거나(멀티스레드, 비동기), 연결당 처리량을 제한해야 한다.

---

## 📌 핵심 정리

```
소켓 상태 진단:
  ss -s: 전체 상태 요약 (timewait 수 즉시 확인)
  ss -tan state <state>: 특정 상태 소켓 목록

TIME_WAIT 관리:
  원인: 활성 종료 후 60초 유지 (정상)
  문제: 포트 고갈 ("Cannot assign requested address")
  해결: Keep-Alive > tcp_tw_reuse=1 > ip_local_port_range 확장
  클라이언트 측만 문제 (서버 listen 포트는 영향 없음)

CLOSE_WAIT = 버그:
  FIN 받았지만 close() 안 한 상태
  → 0이어야 정상, 지속 증가하면 fd 누수

Recv-Q 누적:
  데이터 수신 속도 > 애플리케이션 처리 속도
  → 수신 버퍼 포화 → TCP Zero Window → 송신 중지
  → 애플리케이션 처리 속도 향상 필요

네트워크 처리량 확인:
  sar -n DEV 1: 초당 패킷 수, 처리량 (rxkB/s, txkB/s)
  /proc/net/dev rxdrop: 패킷 드롭 카운터
  → 드롭 발생 시 NIC 용량 증설 또는 로드 분산

핵심 명령어:
  ss -s            → 소켓 상태 요약
  ss -tanp         → 상태별 상세 + 프로세스
  sar -n DEV 1     → NIC 처리량 시계열
  /proc/net/dev    → NIC 통계 (드롭, 에러)
  netstat -s | grep -i "time\|retrans"  → TCP 통계
```

---

## 🤔 생각해볼 문제

**Q1.** `ss -s`에서 `timewait=50,000`이고 서비스가 정상 동작 중이다. 이것이 문제인가?

<details>
<summary>해설 보기</summary>

**서버 측에서는** 문제없다. 서버는 항상 같은 포트(80, 8080 등)를 listen하고, TIME_WAIT 소켓의 4-tuple(서버IP:서버포트:클라이언트IP:클라이언트포트)은 서버 listen 포트 사용을 막지 않는다. **클라이언트 측에서는** 문제가 될 수 있다. `ip_local_port_range`(기본 28,231개)를 초과하면 새 연결이 불가능하다. 서비스가 외부 서버에 연결하는 클라이언트 역할이라면 `tcp_tw_reuse=1`과 `ip_local_port_range` 확장을 검토한다.

</details>

**Q2.** Nginx → Spring Boot 연결에서 `CLOSE_WAIT`가 쌓인다. 어느 쪽의 버그인가?

<details>
<summary>해설 보기</summary>

`CLOSE_WAIT`는 수동 종료 측(FIN을 받은 쪽)에 발생한다. Nginx가 먼저 연결을 끊으면(`keepalive_timeout` 초과 등) Spring Boot 쪽에 `CLOSE_WAIT`가 쌓인다. 이것은 Spring Boot가 Nginx의 FIN을 받았지만 소켓을 닫지 않은 것이다. Spring Boot 애플리케이션 코드에서 `close()` 누락이나, 사용 중인 HTTP 클라이언트 라이브러리(RestTemplate 등)의 연결 풀에서 dead connection을 정리하지 않는 것이 원인일 수 있다.

</details>

**Q3.** `sar -n DEV 1`에서 `rxpck/s=1,200,000`이지만 `rxkB/s=10,000`이다. 이 패턴이 나타내는 것은?

<details>
<summary>해설 보기</summary>

평균 패킷 크기가 10,000KB / 1,200,000 = 약 8.5바이트다. 이것은 실제로 불가능한 수치이므로 단위 착오지만, 개념적으로 초당 120만 패킷에 처리량이 10MB/s라면 패킷당 평균 크기가 매우 작다는 의미다. 이런 패턴은 UDP 또는 TCP ACK 같은 소형 패킷이 대량 발생할 때 나타난다. Redis, Kafka 같은 고성능 프로토콜에서 작은 요청/응답이 많을 때도 발생한다. 패킷 처리량(pps) 한계에 먼저 도달할 수 있으므로 NIC offload 설정(TSO, GRO)과 multi-queue NIC 활용을 검토한다.

</details>

---

**[⬅️ 이전: I/O 분석](./03-io-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: strace와 perf ➡️](./05-strace-perf.md)**
