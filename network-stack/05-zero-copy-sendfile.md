# 05. Zero-copy — sendfile()이 복사를 줄이는 방식

## 🎯 핵심 질문

- 일반 파일 전송에서 데이터는 몇 번 복사되는가?
- `sendfile()`은 어떤 복사를 없애고, 어떻게 가능한가?
- Kafka가 `sendfile()`로 Consumer에게 메시지를 전달하는 정확한 흐름은?
- `splice()`, `tee()`, `vmsplice()`는 각각 어떤 상황에서 쓰는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Kafka 브로커가 Consumer에게 수 GB/s의 처리량을 내는 비결 중 하나가 Zero-copy다.** 디스크 세그먼트 파일을 읽어 소켓으로 보낼 때 `sendfile()`을 사용하면 유저 공간으로의 복사가 없다. Page Cache에서 NIC 버퍼로 직접 전달된다. CPU가 데이터 복사에 소비되는 시간이 사라진다.

**Nginx의 `sendfile on` 설정이 정적 파일 서빙에서 성능을 높이는 이유는?** 정적 파일(HTML, CSS, JS, 이미지)을 서빙할 때 `read()` + `write()` 대신 `sendfile()`을 사용해 유저 공간 복사를 제거한다. 특히 파일이 Page Cache에 있을 때 CPU 사용률이 크게 떨어진다.

**Java `FileChannel.transferTo()`가 "Zero-copy"라고 불리는 이유는?** 내부적으로 `sendfile()` 시스템 콜을 사용한다. JVM에서도 OS의 Zero-copy 기능을 활용할 수 있음을 보여준다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```java
// 실수 1: 파일 전송에 read() + write() 반복
FileInputStream fis = new FileInputStream("large_video.mp4");
OutputStream os = socket.getOutputStream();
byte[] buf = new byte[8192];
int n;
while ((n = fis.read(buf)) != -1) {
    os.write(buf, 0, n);
}
// → 4번의 복사 + 2번의 컨텍스트 스위칭
// → 수 GB 파일 전송 시 CPU 바인딩 발생

// Zero-copy 버전
FileChannel fc = new FileInputStream("large_video.mp4").getChannel();
SocketChannel sc = SocketChannel.open(serverAddress);
fc.transferTo(0, fc.size(), sc);  // → 내부적으로 sendfile() 사용
```

```bash
# 실수 2: Nginx sendfile off 상태로 운영
$ cat /etc/nginx/nginx.conf | grep sendfile
sendfile off;  ← 기본값이 off인 경우 (설정 누락)
# 정적 파일 서빙 시 CPU 사용률 불필요하게 높음

# 올바른 설정:
# sendfile on;
# tcp_nopush on;   ← sendfile과 함께 TCP 최적화
# tcp_nodelay on;  ← sendfile 이후 즉시 전송
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# sendfile() 시스템 콜 사용 확인 (strace)
$ strace -e trace=sendfile64 -p $(pgrep nginx) 2>&1 | head -10
sendfile64(12, 7, [0] -> [524288], 524288) = 524288
# 인자: 소켓fd, 파일fd, 오프셋, 크기
# 반환값: 전송된 바이트 수

# Kafka의 sendfile 확인
$ strace -e trace=sendfile,sendfile64 -p $(pgrep kafka) 2>&1 | head -5
sendfile64(23, 19, [0], 1048576) = 1048576

# CPU 사용률 비교 (sendfile on/off)
# sendfile off: 파일 서빙 중 CPU ~40%
# sendfile on:  파일 서빙 중 CPU ~5% (8배 감소!)
$ mpstat 1 | grep -v "^$"
```

---

## 🔬 내부 동작 원리

### 일반 파일 전송의 4단계 복사

```
일반 read() + write() 파일 전송:

[디스크]
    │ ① DMA 복사 (디스크 → 커널 Page Cache)
    ▼
[커널 Page Cache]  ← sk_buff 아님, 파일 캐시
    │ ② CPU 복사 (Page Cache → 유저 버퍼)
    ▼
[유저 공간 버퍼]   ← byte[] buf in Java
    │ ③ CPU 복사 (유저 버퍼 → 커널 소켓 송신 버퍼)
    ▼
[커널 소켓 송신 버퍼] (sk_buff)
    │ ④ DMA 복사 (소켓 버퍼 → NIC)
    ▼
[NIC → 네트워크]

컨텍스트 스위칭:
  read() 호출 시: 유저→커널 전환
  read() 반환 시: 커널→유저 전환
  write() 호출 시: 유저→커널 전환
  write() 반환 시: 커널→유저 전환
  총 4번의 컨텍스트 스위칭 + CPU가 ②③ 복사 수행

총 복사: 4번 (DMA 2 + CPU 2)
CPU 관여: ② + ③ (CPU 바운드 복사)
```

### sendfile()의 Zero-copy

```
sendfile(out_sock_fd, in_file_fd, offset, count) 사용:

[디스크]
    │ ① DMA 복사 (디스크 → 커널 Page Cache)
    ▼
[커널 Page Cache]
    │   SG(Scatter-Gather) DMA 지원 NIC이 있으면:
    │   파일 페이지 디스크립터만 소켓에 전달 (복사 없음!)
    │   NIC이 직접 Page Cache에서 읽어 전송
    │
    │   SG DMA 미지원 시:
    │ ② CPU 복사 (Page Cache → NIC 버퍼)
    ▼
[NIC → 네트워크]

컨텍스트 스위칭:
  sendfile() 호출 시: 유저→커널 전환
  sendfile() 반환 시: 커널→유저 전환
  총 2번 (read+write의 절반!)

유저 공간 이동 없음: 유저 버퍼 복사 완전 제거

SG DMA 지원 시: 총 복사 2번 (DMA 1 + DMA 1)
SG DMA 미지원 시: 총 복사 3번 (DMA 1 + CPU 1 + DMA 1)
```

### sendfile() 커널 구현

```c
/* sendfile64 시스템 콜 핵심 로직 (단순화) */
ssize_t do_sendfile(int out_fd, int in_fd, loff_t *ppos, size_t count) {
    /* in_fd: 파일 fd, out_fd: 소켓 fd */
    
    /* 파일의 Page Cache 페이지 참조 */
    struct page **pages = get_pages(in_file, offset, count);
    
    /* 소켓에 페이지 참조 추가 (복사 없음!) */
    /* SG DMA 지원 NIC: 페이지 디스크립터만 NIC에 전달 */
    sock->ops->sendpage(sock, pages, count);
    
    /* NIC이 DMA로 Page Cache에서 직접 읽어 전송 */
    return sent;
}
```

### Kafka의 sendfile() 활용

```
Kafka Consumer fetch 요청 처리:

Consumer → Kafka Broker: FetchRequest (파티션, 오프셋, 최대 바이트)

Broker 처리:
  1. 로그 세그먼트 파일 위치 계산 (오프셋 인덱스 조회)
  2. 파일 fd 획득

  Java NIO 코드:
  FileChannel.transferTo(position, count, socketChannel)
  → 내부적으로 sendfile64() 시스템 콜

커널 동작:
  [세그먼트 파일 (.log)]
      │ DMA 읽기 → Page Cache (파일이 캐시에 있으면 바로)
      ▼
  [Page Cache]
      │ sendfile: 페이지 디스크립터 → 소켓 버퍼 (복사 없음)
      ▼
  [NIC] → TCP 패킷 전송 → Consumer

Kafka Broker CPU 절감:
  일반 read+write: 초당 1GB 전송 시 CPU ~30%
  sendfile:        초당 1GB 전송 시 CPU ~5%
  → CPU 여유로 파티션 리더 선출, 메타데이터 관리 등에 활용
```

### splice() — 파이프를 이용한 Zero-copy

```
sendfile()은 파일→소켓 전용
splice()는 더 범용적:
  파이프←→파일, 파이프←→소켓 연결 가능

splice(in_fd, in_off, pipe_fd[1], NULL, count, flags)
splice(pipe_fd[0], NULL, out_fd, out_off, count, flags)

사용 예 (파일 → 파이프 → 소켓):
  int pipefd[2];
  pipe(pipefd);
  splice(file_fd, &offset, pipefd[1], NULL, count, SPLICE_F_MOVE);
  splice(pipefd[0], NULL, sock_fd, NULL, count, SPLICE_F_MOVE);

tee() — 파이프 데이터 복제 (읽지 않고 복사):
  tee(pipe1_read, pipe2_write, count, 0)
  → 로깅: 동일 데이터를 파일 + 소켓으로 분기

vmsplice() — 유저 공간 메모리를 파이프로 (Zero-copy):
  iov.iov_base = user_buf;
  iov.iov_len = count;
  vmsplice(pipefd[1], &iov, 1, SPLICE_F_GIFT)
  → 유저 버퍼를 커널이 소유 (복사 없음, 이후 user_buf 접근 금지)
```

### io_uring — 차세대 Zero-copy

```
io_uring (Linux 5.1+):
  비동기 I/O 인터페이스 + Zero-copy 지원
  시스템 콜 오버헤드 최소화 (공유 메모리 ring buffer 사용)

  IORING_OP_SENDMSG_ZC (Zero-copy send, Linux 6.0+):
    유저 버퍼를 커널에 등록 (등록 1회)
    이후 send 시 복사 없이 등록된 메모리 참조

  현재 지원:
    Nginx io_uring 실험적 지원
    io_uring 기반 파일 서빙 (sendfile 대체)
```

---

## 💻 실전 실험

### 실험 1: sendfile vs read+write 성능 비교

```c
/* sendfile_bench.c */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/sendfile.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>
#include <time.h>

#define FILE_SIZE (512 * 1024 * 1024)  // 512MB

long now_ms() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000 + ts.tv_nsec / 1000000;
}

void send_with_read_write(int file_fd, int sock_fd) {
    char buf[65536];
    int n;
    lseek(file_fd, 0, SEEK_SET);
    while ((n = read(file_fd, buf, sizeof(buf))) > 0) {
        write(sock_fd, buf, n);
    }
}

void send_with_sendfile(int file_fd, int sock_fd) {
    off_t offset = 0;
    sendfile(sock_fd, file_fd, &offset, FILE_SIZE);
}

// main()은 서버/클라이언트 소켓 셋업 후 각각 측정
```

```bash
$ gcc sendfile_bench.c -o sendfile_bench
# 서버 터미널: ./sendfile_bench server
# 클라이언트 터미널: ./sendfile_bench client
# 예상 결과 (512MB 파일, localhost):
# read+write: 1850ms, CPU 사용률 ~35%
# sendfile:    420ms, CPU 사용률 ~5%
```

### 실험 2: strace로 Nginx sendfile 확인

```bash
$ nginx -s start  # sendfile on 설정
$ PID=$(pgrep nginx | tail -1)  # worker 프로세스

# sendfile 호출 추적
$ strace -e trace=sendfile64 -p $PID 2>&1 &

# 클라이언트에서 정적 파일 요청
$ curl http://localhost/large_image.jpg > /dev/null

# strace 출력:
# sendfile64(5, 8, [0] -> [1048576], 1048576) = 1048576
# ↑ 소켓fd  ↑ 파일fd  ↑오프셋     ↑크기       ↑전송바이트

# CPU 사용률 비교
$ nginx -s stop
# nginx.conf: sendfile off
$ nginx -s start
$ ab -n 1000 -c 100 http://localhost/large_image.jpg
# sendfile off: CPU ~40%, 처리량 X MB/s
# sendfile on:  CPU ~5%,  처리량 Y MB/s (Y > X)
```

### 실험 3: Java FileChannel.transferTo() 시스템 콜 확인

```java
// TransferToDemo.java
import java.io.*;
import java.net.*;
import java.nio.*;
import java.nio.channels.*;

public class TransferToDemo {
    public static void main(String[] args) throws Exception {
        // 서버 소켓
        ServerSocketChannel server = ServerSocketChannel.open();
        server.bind(new InetSocketAddress(9990));
        SocketChannel client = server.accept();

        // 파일 전송 (transferTo = sendfile 사용)
        FileChannel fc = new FileInputStream("/tmp/large_file").getChannel();
        long start = System.currentTimeMillis();
        long transferred = fc.transferTo(0, fc.size(), client);
        System.out.println("전송: " + transferred + " bytes in " +
            (System.currentTimeMillis() - start) + "ms");
    }
}
```

```bash
# strace로 sendfile 시스템 콜 확인
$ strace -e trace=sendfile,sendfile64 -p $(pgrep java) 2>&1
# sendfile64(5, 7, [0] -> [512000000], 512000000) = 512000000
# ↑ Java FileChannel.transferTo()가 내부적으로 sendfile() 사용!
```

---

## 📊 성능/비용 비교

**파일 전송 방식별 비교 (512MB 파일, 1Gbps LAN)**:

```
방식                 CPU 사용률  전송 시간  메모리 복사
read()+write()          35%      1850ms    4회 (DMA2+CPU2)
sendfile() (SG DMA)      5%       420ms    2회 (DMA2만)
sendfile() (no SG DMA)  15%       480ms    3회 (DMA2+CPU1)
io_uring ZC send         3%       380ms    1회 (DMA1)

SG DMA = Scatter-Gather DMA 지원 NIC (대부분 현대 서버 NIC)
```

**Kafka throughput vs CPU 사용률**:

```
설정                   Throughput    CPU 사용률
sendfile (기본)         800 MB/s      15%
read+write              800 MB/s      50%  ← CPU 병목
read+write (더 많이)    480 MB/s      100% ← CPU 포화

결론: sendfile로 CPU 여유를 확보해 더 많은 파티션/프로세싱 처리 가능
```

---

## ⚖️ 트레이드오프

**sendfile()의 제약**

`sendfile()`은 파일에서 소켓으로만 동작한다. 데이터를 변환하거나(암호화, 압축), 헤더를 추가해야 한다면 유저 공간을 거쳐야 한다. Nginx의 gzip 압축이나 TLS 암호화는 `sendfile()`을 사용하지 못하고 일반 read/write 경로를 사용한다. `sendfile_max_chunk` 설정으로 한 번에 전송할 최대 크기를 제한해 다른 연결이 starve되지 않게 조절할 수 있다.

**io_uring의 등장**

`sendfile()`은 파일→소켓 전용이지만, `io_uring`의 Zero-copy send는 유저 버퍼에서 소켓으로도 복사 없이 전송이 가능하다. 아직 실험적이지만, 미래에는 `sendfile()`의 제약을 극복한 범용 Zero-copy가 가능해진다.

---

## 📌 핵심 정리

```
일반 read()+write():
  4번 복사: 디스크→PageCache, PageCache→유저버퍼, 유저버퍼→소켓버퍼, 소켓버퍼→NIC
  4번 컨텍스트 스위칭

sendfile():
  2번 복사: 디스크→PageCache, PageCache→NIC (SG DMA 지원 시)
  2번 컨텍스트 스위칭
  유저 공간 이동 없음 → CPU 데이터 복사 없음

사용처:
  Nginx: sendfile on; (정적 파일 서빙)
  Kafka: FileChannel.transferTo() (Consumer 전달)
  Java: FileChannel.transferTo() → 내부 sendfile()
  Apache httpd: EnableSendfile on

제약:
  파일→소켓 전용 (소켓→소켓, 유저버퍼→소켓 불가)
  데이터 변환(암호화, 압축) 불가
  TLS (HTTPS)에서는 sendfile 사용 불가 (암호화 필요)

Nginx 권장 설정:
  sendfile on;
  tcp_nopush on;    ← sendfile 후 TCP_CORK (큰 청크 전송)
  tcp_nodelay on;   ← TCP_CORK 해제 후 즉시 전송

핵심 진단:
  strace -e sendfile64 -p <pid>  → sendfile 호출 확인
  mpstat / top                   → sendfile 전후 CPU 사용률 비교
```

---

## 🤔 생각해볼 문제

**Q1.** Nginx가 HTTPS(TLS)를 처리할 때 `sendfile on`이 설정돼 있어도 실제로 `sendfile()`을 사용하지 않는다. 왜인가?

<details>
<summary>해설 보기</summary>

TLS 암호화는 유저 공간에서 수행된다. 파일 데이터가 커널 Page Cache에 있더라도, 암호화 라이브러리(OpenSSL 등)가 유저 공간에서 데이터를 가져와 암호화한 후 소켓에 써야 한다. `sendfile()`은 유저 공간을 거치지 않으므로 암호화를 수행할 기회가 없다. 따라서 HTTPS 서버에서는 `sendfile on`이 있어도 실제로는 `read()+write()` 경로로 동작한다. `ssl_sendfile on`은 TLS offload 하드웨어가 있을 때만 의미가 있다.

</details>

**Q2.** Kafka Consumer가 매우 빠르게 메시지를 소비하면 Broker의 `sendfile()`이 디스크 I/O 없이 동작할 수 있는가?

<details>
<summary>해설 보기</summary>

**그렇다.** Producer가 방금 쓴 메시지는 Page Cache에 있다. Consumer가 즉시 소비하면 `sendfile()`이 Page Cache에서 직접 NIC으로 전달한다. 이 경우 디스크 I/O가 전혀 없다. Kafka가 "메시지가 디스크에 저장되지만 빠르다"는 역설이 가능한 이유가 여기에 있다. Consumer lag이 작을수록 Page Cache 히트율이 높아 디스크 I/O가 적다. Consumer가 뒤처질수록(Consumer lag 큼) Page Cache에서 밀려난 이전 메시지를 디스크에서 읽어야 하므로 처리량이 감소한다.

</details>

**Q3.** `splice()`를 사용해 소켓에서 파이프를 거쳐 다른 소켓으로 데이터를 전달할 수 있다. 이것이 L7 프록시(Nginx reverse proxy)와 다른 점은?

<details>
<summary>해설 보기</summary>

`splice()`를 사용한 전달은 데이터가 유저 공간을 거치지 않아 복사 비용이 없다. 하지만 커널이 데이터를 보지 않으므로 HTTP 헤더 수정, URL 재작성, 인증, 로깅 등의 L7 처리가 불가능하다. Nginx의 reverse proxy는 유저 공간에서 HTTP 요청을 파싱하고, 헤더를 수정하고(`X-Forwarded-For` 추가 등), 응답을 클라이언트에게 전달한다. 이 과정에서 전통적 복사가 발생하지만 L7 처리가 가능하다. 데이터 변환 없이 단순 전달만 필요한 TCP 프록시(HAProxy TCP mode)는 `splice()`로 Zero-copy를 구현할 수 있다.

</details>

---

**[⬅️ 이전: 커널 네트워크 파라미터 튜닝](./04-kernel-network-tuning.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — Linux Namespace ➡️](../cgroups-namespace/01-linux-namespace.md)**
