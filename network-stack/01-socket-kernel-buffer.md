# 01. 소켓과 커널 버퍼 — send/recv의 실체

## 🎯 핵심 질문

- `send()` 호출 시 데이터는 실제로 언제 네트워크로 전송되는가?
- 커널 송신/수신 버퍼(`sk_buff`)는 어디에 있고 어떻게 동작하는가?
- 수신 버퍼가 가득 찼을 때 TCP 흐름 제어(Zero Window)는 어떻게 발동되는가?
- `ss -tmn`으로 소켓 버퍼 상태를 어떻게 해석하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**`send()` 반환값이 성공이어도 네트워크 장애가 발생하면 데이터가 유실될 수 있다.** `send()`는 커널 송신 버퍼에 복사가 완료됐을 때 반환한다. 실제 네트워크 전송과 ACK 수신은 그 이후다. TCP가 재전송을 보장하지만, 소켓이 닫히거나 애플리케이션이 종료되면 버퍼에 남아있던 데이터는 사라진다.

**Redis 서버와 클라이언트 사이에서 응답이 갑자기 느려진다.** 수신 버퍼가 가득 차서 TCP Zero Window가 발생한 것이다. 클라이언트가 응답을 처리하지 못해 수신 버퍼가 차면, TCP가 Zero Window Advertisement를 보내 송신자를 멈춘다. `ss -tmn`에서 `rcv-buf`가 꽉 찬 소켓을 확인할 수 있다.

**gRPC 스트리밍에서 back-pressure가 작동하지 않아 OOM이 발생했다.** 수신측이 느리면 커널 수신 버퍼가 가득 차고, TCP 흐름 제어가 송신을 늦춘다. 하지만 애플리케이션 레벨 back-pressure가 없으면 송신측이 커널 버퍼를 계속 채워 결국 메모리 부족이 발생한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```java
// 실수 1: send() 성공 = 전송 완료라고 오해
Socket s = new Socket("server", 8080);
OutputStream os = s.getOutputStream();
os.write(data);  // ← 커널 버퍼에 복사됐을 뿐
os.flush();      // ← 여전히 커널 버퍼에서 전송 대기 중
s.close();       // ← 버퍼 데이터가 전송 완료되기 전에 닫힘 가능!
// TCP RST 전송 → 상대방이 데이터를 못 받을 수 있음

// 올바른 처리: SO_LINGER 설정 또는 애플리케이션 레벨 ACK 확인

// 실수 2: 수신 버퍼 크기 무시
byte[] buf = new byte[1];  // 1바이트씩 읽기
while ((n = is.read(buf)) != -1) {
    // → 수신 버퍼에 데이터가 있어도 1바이트씩 읽음
    // → read() 시스템 콜 폭발 (n바이트 → n번 시스템 콜)
}
```

```bash
# 실수 3: 소켓 버퍼 포화를 인지 못 함
$ ss -tmn  # 확인 안 함
# 실제로는:
# Recv-Q 87380  ← 수신 버퍼 가득 참
# TCP Zero Window 발생 중
# → 송신측이 대기 중인데 "왜 느리지?" 의문만 가짐
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 소켓 버퍼 상태 실시간 확인
$ ss -tmn
State    Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
ESTAB    0       0       10.0.0.1:8080       10.0.0.2:54321
ESTAB    87380   0       10.0.0.1:8080       10.0.0.3:54322  ← 수신 버퍼 포화!
ESTAB    0       327680  10.0.0.1:8080       10.0.0.4:54323  ← 송신 버퍼 포화!

# Recv-Q > 0: 애플리케이션이 데이터를 읽는 속도 < 데이터 수신 속도
# Send-Q > 0: 커널이 전송하는 속도 < 애플리케이션이 write() 속도

# 소켓 버퍼 크기 확인
$ cat /proc/sys/net/core/rmem_default   # 수신 버퍼 기본 크기
87380
$ cat /proc/sys/net/core/wmem_default   # 송신 버퍼 기본 크기
16384

# 버퍼 크기 튜닝 (고대역폭 환경)
$ sysctl net.core.rmem_max=134217728    # 최대 128MB
$ sysctl net.core.wmem_max=134217728
$ sysctl net.ipv4.tcp_rmem="4096 87380 134217728"
$ sysctl net.ipv4.tcp_wmem="4096 16384 134217728"
```

---

## 🔬 내부 동작 원리

### send() 시스템 콜 경로

```
send(sockfd, buf, len, 0) 호출:

[애플리케이션 버퍼]  "Hello, World" (유저 공간)
         │
         │ send() 시스템 콜
         ▼
[커널: sock_sendmsg()]
         │
         ▼
[tcp_sendmsg()]
  1. 소켓 잠금 획득
  2. sk_buff 구조체 할당
  3. 유저 버퍼 → sk_buff로 복사 (copy_from_user)
  4. TCP 헤더 추가 (시퀀스 번호, 포트, 체크섬)
  5. 송신 큐(sk_write_queue)에 sk_buff 추가
  6. 반환값 = 복사된 바이트 수
         │
         │ ★ 여기서 send()가 반환! 실제 전송은 아직 안 됨
         │
         ▼ (비동기)
[tcp_output / tcp_write_xmit()]
  네트워크 상태 확인 (혼잡 제어, 흐름 제어)
  → 전송 가능하면: sk_buff → IP 레이어 → NIC → 전송
  → 전송 불가하면: 대기 (Send-Q에서 대기)

send() 블로킹 조건:
  송신 버퍼(sk_sndbuf)가 가득 찼을 때
  → Blocking 소켓: 공간이 생길 때까지 블로킹
  → Non-Blocking 소켓: EAGAIN 반환
```

### sk_buff (Socket Buffer) 구조

```
sk_buff = 커널에서 네트워크 패킷을 표현하는 구조체

[sk_buff]
  ├── data: 패킷 데이터 포인터
  ├── len:  패킷 길이
  ├── next/prev: 연결 리스트 포인터
  ├── sk:   소속 소켓 포인터
  ├── protocol: 프로토콜 (TCP, UDP 등)
  └── [헤더 공간 + 데이터 + 테일 공간]

헤더 추가 방식 (Zero-copy 최적화):
  ┌──────────┬──────────────────────┬──────┐
  │ 헤더 공간  │ 실제 데이터             │ 테일  │
  │ (미리 예약)│                      │      │
  └──────────┴──────────────────────┴──────┘
  TCP 헤더 추가: 헤더 공간에 직접 기록 (복사 없음!)
  IP 헤더 추가: 헤더 공간에 직접 기록 (복사 없음!)
  → 데이터 자체는 복사 없이 헤더만 앞에 추가

소켓 수신/송신 버퍼:
  sk_rcvbuf: 수신 버퍼 최대 크기
  sk_sndbuf: 송신 버퍼 최대 크기
  실제 sk_buff 체인이 이 크기를 채움
```

### TCP 흐름 제어 — Zero Window

```
수신측 버퍼가 가득 찼을 때:

[수신측 커널 수신 버퍼]
┌────────────────────────────────────┐
│ 가득 참! (87380 bytes)               │ ← 애플리케이션이 읽지 않아 꽉 참
└────────────────────────────────────┘

TCP ACK 패킷에 Window Size = 0 광고:
  "나 더 이상 받을 공간 없어. 그만 보내!"

[송신측 반응]
  송신 중지 (Zero Window 상태)
  → 주기적으로 Zero Window Probe 전송
  → 수신측 버퍼에 공간이 생기면 Window Update 수신
  → 전송 재개

ss -tmn에서 확인:
  State    Recv-Q  Send-Q
  ESTAB    87380   0       ← 수신측: 수신 버퍼 가득 (Zero Window 광고 중)
  ESTAB    0       327680  ← 송신측: 송신 버퍼에 데이터 대기 중 (Zero Window 수신)

실무 영향:
  수신측 애플리케이션이 데이터를 빨리 읽지 못하면
  → 수신 버퍼 포화 → Zero Window → 송신 중지
  → 송신측 애플리케이션도 send() 블로킹
  → 양측 모두 응답 느려짐
```

### recv() 시스템 콜 경로

```
NIC로 패킷 수신 시:

1. NIC → DMA → 커널 메모리 (sk_buff 생성)
2. NIC → CPU 인터럽트
3. 인터럽트 핸들러: sk_buff → 네트워크 스택 처리
4. IP 계층: 역캡슐화 → TCP 세그먼트 추출
5. TCP 계층: 순서 재조합, ACK 전송
6. sk_buff를 소켓 수신 큐(sk_receive_queue)에 추가
7. 수신 대기 중인 프로세스 Wake-up

recv(sockfd, buf, len, 0) 호출:

  수신 큐에 데이터 있음:
    sk_buff 데이터 → 유저 버퍼 복사 (copy_to_user)
    반환값 = 복사된 바이트 수

  수신 큐 비어있음 + Blocking:
    프로세스 Sleep → 데이터 도착 → Wake-up → 복사
```

---

## 💻 실전 실험

### 실험 1: 소켓 버퍼 상태 실시간 모니터링

```bash
# ss로 소켓 상세 정보 확인
$ ss -tmni  # -i: TCP 내부 정보 포함

State    Recv-Q  Send-Q  Local:Port  Peer:Port
ESTAB    0       0       10.0.0.1:8080  10.0.0.2:54321
         cubic wscale:9,7 rto:204 rtt:4.123/1.234 ato:40
         mss:1448 pmtu:1500 rcvmss:1448 advmss:1448
         cwnd:10 ssthresh:14 bytes_sent:123456 bytes_acked:123456
         segs_out:100 segs_in:100 data_segs_out:50 data_segs_in:50
         send 28.1Mbps lastsnd:1 lastrcv:1 lastack:1
         pacing_rate 33.7Mbps delivery_rate 28.1Mbps
         rcv_rtt:100 rcv_space:87380 rcv_ssthresh:87380
         minrtt:3.912

# 핵심 필드:
# Recv-Q: 수신 버퍼에 쌓인 바이트 (미처리 데이터)
# Send-Q: 송신 버퍼에 대기 중인 바이트 (미전송 데이터)
# cwnd: 혼잡 제어 윈도우 (이 크기만큼만 전송 가능)
# rcv_space: 상대방이 광고한 수신 창 크기

# TIME_WAIT 포함 전체 소켓 상태 요약
$ ss -s
Total: 500
TCP:   400 (estab 300, closed 50, orphaned 10, timewait 40)
```

### 실험 2: Zero Window 발생 재현

```python
# zero_window_demo.py
# 서버: 수신하지만 읽지 않음 → 수신 버퍼 포화
import socket, time, threading

def slow_server():
    s = socket.socket()
    s.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 8192)  # 작은 수신 버퍼
    s.bind(('', 9999))
    s.listen(1)
    conn, _ = s.accept()
    print("연결 수락. 데이터 받지만 읽지 않음...")
    time.sleep(30)  # 데이터를 읽지 않음
    conn.close()

def fast_sender():
    time.sleep(0.5)
    s = socket.socket()
    s.connect(('localhost', 9999))
    data = b'X' * 65536
    total = 0
    while True:
        try:
            n = s.send(data)
            total += n
            print(f"보낸 바이트: {total}")
        except Exception as e:
            print(f"send 차단: {e}")
            break

threading.Thread(target=slow_server).start()
threading.Thread(target=fast_sender).start()
```

```bash
$ python3 zero_window_demo.py &
# 별도 터미널에서:
$ ss -tmn | grep 9999
# Send-Q가 점점 차다가 멈춤 → Zero Window 확인
$ ss -tmni | grep "rcv_space:0"
# rcv_space=0 → Zero Window 상태
```

### 실험 3: 소켓 버퍼 크기 최적화 실험

```bash
# 고대역폭 환경에서 버퍼 크기와 처리량 관계
# BDP (Bandwidth-Delay Product) = 대역폭 * RTT
# 예: 1Gbps, RTT 100ms → BDP = 12.5MB
# → 버퍼를 BDP 이상으로 설정해야 파이프 가득 채울 수 있음

# 현재 버퍼 설정
$ sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
net.ipv4.tcp_rmem = 4096 87380 6291456   # min/default/max
net.ipv4.tcp_wmem = 4096 16384 4194304

# 100Mbps, RTT 50ms: BDP = 625KB → 기본값 87380으로 충분
# 10Gbps, RTT 50ms: BDP = 62.5MB → 버퍼 부족!

# 고대역폭 튜닝
$ sysctl -w net.ipv4.tcp_rmem="4096 131072 134217728"
$ sysctl -w net.ipv4.tcp_wmem="4096 131072 134217728"
$ sysctl -w net.core.rmem_max=134217728
$ sysctl -w net.core.wmem_max=134217728
# tcp_mem: 자동 조정 허용
$ sysctl -w net.ipv4.tcp_mem="65536 131072 262144"
```

---

## 📊 성능/비용 비교

| 항목 | Recv-Q > 0 | Send-Q > 0 |
|------|-----------|-----------|
| 의미 | 애플리케이션이 데이터를 처리 못 함 | 커널이 데이터를 전송 못 함 |
| 원인 | 처리 느림, 블로킹 코드, CPU 부족 | 네트워크 느림, Zero Window, 혼잡 |
| 영향 | Zero Window → 송신 중지 | send() 블로킹 → 애플리케이션 대기 |
| 해결 | 처리 속도 향상, 버퍼 증가 | 네트워크 대역폭 확인, 버퍼 증가 |

**소켓 버퍼 크기와 처리량 관계 (BDP 기준)**:

```
네트워크 조건               최적 버퍼 크기
1Gbps, RTT 1ms   BDP=125KB  → 기본값 87380으로 거의 충분
1Gbps, RTT 50ms  BDP=6.25MB → 버퍼 6MB 이상 필요
10Gbps, RTT 50ms BDP=62.5MB → 버퍼 64MB+ 필요

버퍼 부족 시:
  파이프가 절반만 채워짐 → 처리량 50% 이하
  TCP "slow" 전송 (ACK 기다림)
```

---

## ⚖️ 트레이드오프

**소켓 버퍼 크기 증가의 비용**

버퍼를 크게 설정하면 대역폭 활용률이 올라가지만 메모리 소비가 증가한다. 연결 수 × 버퍼 크기가 서버 메모리를 압박할 수 있다. 10,000개 연결 × 128MB 버퍼는 불가능하다. TCP 자동 튜닝(`tcp_moderate_rcvbuf=1`, 기본 활성화)을 사용하면 커널이 RTT와 대역폭을 측정해 버퍼를 자동 조정한다.

**SO_LINGER와 데이터 안전성**

`close()` 호출 시 송신 버퍼에 데이터가 남아있으면 기본 동작은 백그라운드에서 전송을 완료하는 것이다. 하지만 프로세스가 즉시 종료되면 TCP RST가 전송되어 상대방이 데이터를 못 받을 수 있다. `SO_LINGER`로 close() 시 최대 대기 시간을 설정하거나, 애플리케이션 레벨 ACK를 구현해야 완전한 데이터 전달을 보장할 수 있다.

---

## 📌 핵심 정리

```
send() = 커널 송신 버퍼로 복사 완료 후 반환
  → 실제 전송은 커널이 비동기로 처리
  → Send-Q: 아직 전송 안 된 데이터 크기

recv() = 커널 수신 버퍼에서 유저 버퍼로 복사
  → Recv-Q: 아직 애플리케이션이 읽지 않은 데이터

TCP Zero Window:
  수신 버퍼 포화 → Window Size = 0 광고
  → 송신측 전송 중지 → Send-Q 증가
  → 애플리케이션이 느리게 읽는 것이 원인

소켓 버퍼 최적화:
  BDP = 대역폭 * RTT → 이것보다 버퍼가 커야 파이프 가득
  tcp_rmem/wmem으로 설정 (min/default/max)

핵심 진단 명령어:
  ss -tmn          → Recv-Q/Send-Q 현황
  ss -tmni         → TCP 내부 상태 (cwnd, rcv_space 등)
  ss -s            → 전체 소켓 통계 요약
  /proc/net/sockstat → 커널 소켓 통계
```

---

## 🤔 생각해볼 문제

**Q1.** Redis 클라이언트 라이브러리가 `pipeline()`을 사용해 100개의 명령어를 한 번에 보낸다. 이것이 send() 시스템 콜을 몇 번 호출하는지와 실제 TCP 패킷 수는 같은가?

<details>
<summary>해설 보기</summary>

같지 않다. 클라이언트가 100개 명령어를 `send()` 1번에 보내도, TCP는 MSS(Maximum Segment Size, 보통 1448바이트) 단위로 분할해 여러 패킷으로 전송할 수 있다. 반대로 여러 번의 `send()`가 Nagle 알고리즘에 의해 하나의 패킷으로 합쳐질 수도 있다. 시스템 콜 횟수 ≠ 패킷 수다. `strace -e trace=send`로 시스템 콜을, `tcpdump`로 실제 패킷을 각각 확인할 수 있다.

</details>

**Q2.** `ss -tmn`에서 `Send-Q=327680`이다. 이 데이터는 언제 전송되는가?

<details>
<summary>해설 보기</summary>

상대방의 TCP 수신 창(Receive Window)이 0보다 크고 혼잡 제어 창(cwnd)이 허용하는 범위 안에서 커널이 자동으로 전송한다. `Send-Q`가 높다면 상대방이 Zero Window를 광고했거나, 네트워크 혼잡으로 cwnd가 제한됐을 가능성이 높다. 상대방이 데이터를 읽어 Window Update를 보내면 커널이 자동으로 나머지를 전송한다. 애플리케이션이 개입할 필요가 없다.

</details>

**Q3.** 소켓을 `close()` 했는데 `ss`에서 해당 연결이 아직 보인다. 어떤 상태이며 왜 그런가?

<details>
<summary>해설 보기</summary>

`TIME_WAIT` 또는 `CLOSE_WAIT` 상태일 가능성이 높다. `TIME_WAIT`는 활성 종료 측(먼저 FIN을 보낸 쪽)이 일정 시간(2 * MSL, 보통 60초) 동안 유지한다. 이는 지연된 패킷이 새 연결과 혼동되지 않도록 하기 위함이다. `CLOSE_WAIT`는 상대방이 FIN을 보냈는데 애플리케이션이 아직 `close()`를 호출하지 않은 상태다. 이것이 쌓이면 fd 누수나 커넥션 풀 고갈의 신호다.

</details>

---

**[홈으로 🏠](../README.md)** | **[다음: TCP 소켓 상태와 Accept Queue ➡️](./02-tcp-socket-state.md)**
