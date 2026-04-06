# 03. Non-Blocking I/O와 Polling — busy-wait의 함정

## 🎯 핵심 질문

- `O_NONBLOCK`을 설정하면 `read()`의 동작이 어떻게 달라지는가?
- `EAGAIN`은 무엇이고 애플리케이션은 이를 어떻게 처리해야 하는가?
- busy-wait이 CPU를 낭비하는 원리는 무엇인가?
- Non-Blocking I/O 단독으로는 왜 고성능 서버를 만들 수 없는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Netty, Node.js, Redis의 이벤트 루프 모두 Non-Blocking fd를 기반으로 동작한다.** `epoll`과 같은 I/O 멀티플렉싱이 Non-Blocking fd와 조합될 때 비로소 의미를 갖는다. Non-Blocking의 동작 원리를 모르면 이벤트 루프가 왜 그렇게 설계됐는지 이해할 수 없다.

**Non-Blocking I/O만 쓰면 CPU가 100%가 된다.** `EAGAIN`을 받으면 즉시 다시 `read()`를 호출하는 루프(busy-wait)가 가장 단순한 구현이지만, 데이터가 없는 동안 CPU 코어 하나를 100% 소비한다. `select()`/`poll()`/`epoll`이 이 문제를 해결하기 위해 등장했다.

**스트리밍 업로드 중 `EAGAIN`이 발생했다는 에러 로그를 봤다.** 이것은 에러가 아니다. 소켓 수신 버퍼가 일시적으로 비었을 때 Non-Blocking `read()`가 반환하는 정상 상태다. 이를 에러로 처리해 연결을 끊으면 안 된다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```c
// 실수 1: EAGAIN을 에러로 처리
int fd = open_nonblocking_socket();
char buf[4096];
ssize_t n = read(fd, buf, sizeof(buf));
if (n < 0) {
    perror("read failed");  // ← EAGAIN도 에러로 출력
    close(fd);              // ← 정상 소켓을 닫아버림!
    return -1;
}
// 올바른 처리:
if (n < 0 && errno == EAGAIN) {
    // 데이터 없음, 나중에 다시 시도
    return 0;
}

// 실수 2: busy-wait으로 Non-Blocking 구현
while (1) {
    n = read(fd, buf, sizeof(buf));
    if (n > 0) { process(buf, n); break; }
    // EAGAIN 시 즉시 재시도 → CPU 100% 소비
}
// 올바른 접근: epoll/poll로 이벤트 통지 대기
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```c
/* Non-Blocking + poll() 조합 (epoll이 더 효율적, 비교용) */
#include <poll.h>
#include <fcntl.h>
#include <errno.h>

int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

void handle_connection(int fd) {
    set_nonblocking(fd);
    char buf[4096];

    struct pollfd pfd = { .fd = fd, .events = POLLIN };

    while (1) {
        // 이벤트 대기 (CPU 양보)
        int ret = poll(&pfd, 1, 5000);  // 5초 타임아웃
        if (ret == 0) { /* 타임아웃 */ continue; }
        if (ret < 0) { break; }

        if (pfd.revents & POLLIN) {
            ssize_t n = read(fd, buf, sizeof(buf));
            if (n > 0) { process(buf, n); }
            else if (n == 0) { break; /* 연결 종료 */ }
            else if (errno != EAGAIN) { break; /* 실제 오류 */ }
            // EAGAIN: poll이 통지했는데 EAGAIN → 엣지 케이스, 무시
        }
    }
}
```

---

## 🔬 내부 동작 원리

### O_NONBLOCK 설정과 동작 변화

```
Blocking fd (기본):
  read() 호출 → 데이터 없음 → 프로세스 Sleep → 데이터 도착 → Wake-up → 반환
  특성: 호출 시 항상 데이터를 얻거나 에러를 받음

Non-Blocking fd (O_NONBLOCK 설정 후):
  read() 호출 → 데이터 없음 → 즉시 -1 반환 + errno=EAGAIN
  특성: 항상 즉시 반환, 데이터 없으면 EAGAIN

O_NONBLOCK 설정 방법:
  방법 1: open() 시 플래그 지정
    int fd = open("file", O_RDONLY | O_NONBLOCK);

  방법 2: 이미 열린 fd에 설정 (소켓에서 일반적)
    int flags = fcntl(fd, F_GETFL, 0);  // 현재 플래그 읽기
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);  // 추가

  방법 3: 소켓 생성 시
    int sock = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
```

### EAGAIN의 커널 처리 경로

```
Non-Blocking read() 호출 시:

[커널 sys_read → sock_recvmsg → tcp_recvmsg]

  소켓 수신 버퍼 확인:
       │
       ├── 데이터 있음: 복사 후 반환 (정상)
       │
       └── 데이터 없음:
              │
              │  Blocking fd면:
              │    wait_for_data() → 프로세스 Sleep
              │
              └── Non-Blocking fd면:
                   return -EAGAIN  ← 즉시 반환

EAGAIN vs EWOULDBLOCK:
  리눅스에서 EAGAIN == EWOULDBLOCK (같은 값, 11)
  일부 다른 UNIX에서는 다를 수 있음
  → errno == EAGAIN || errno == EWOULDBLOCK 둘 다 체크 권장

EAGAIN이 발생하는 상황:
  1. 소켓 수신 버퍼 비어있음 (가장 흔함)
  2. 소켓 송신 버퍼 가득 참 (write() 시)
  3. Non-Blocking accept() 시 대기 연결 없음
  4. Non-Blocking connect() 시 연결 진행 중
```

### busy-wait이 CPU를 낭비하는 원리

```
busy-wait 구현:

while (1) {
    n = read(fd, buf, sizeof(buf));
    if (n > 0) break;
    if (n < 0 && errno == EAGAIN) continue;  // 즉시 재시도!
}

타임라인:
  t=0ms:  read() → EAGAIN (데이터 없음)
  t=0.001ms: read() → EAGAIN
  t=0.002ms: read() → EAGAIN
  ...
  t=10ms: read() → EAGAIN (1만 번의 루프!)
  t=10ms: [데이터 도착]
  t=10ms: read() → 4096 bytes (성공)

CPU 사용:
  데이터 없는 10ms 동안 CPU 코어 1개를 100% 점유
  → read() 시스템 콜만 1만 번 (오버헤드 큼)
  → 다른 프로세스가 CPU를 쓸 수 없음

비교: Blocking I/O
  t=0ms: read() 호출 → Sleep
  t=10ms: [데이터 도착] → Wake-up
  CPU 사용: 0% (잠든 동안)
  → 데이터 없는 동안 CPU 낭비 없음
```

### Non-Blocking I/O의 올바른 사용 패턴

```
올바른 패턴: 이벤트 통지 + Non-Blocking 읽기

1. fd를 Non-Blocking으로 설정
2. poll/epoll에 fd 등록 (관심 이벤트: POLLIN/EPOLLIN)
3. poll/epoll_wait() 호출 → 이벤트 대기 (CPU 양보!)
4. 이벤트 발생 통지 받음
5. Non-Blocking read() 호출 → 데이터 읽기
6. EAGAIN 받으면 → 2단계로 (더 읽을 데이터 없음)
7. goto 3

Non-Blocking이 반드시 필요한 이유 (Edge Trigger 모드에서):
  epoll ET 모드: 상태 변화 시 딱 1번만 통지
  → 통지 후 EAGAIN까지 모두 읽어야 다음 통지 받음
  → Blocking read()로는 마지막 읽기에서 무한 대기 가능
  → 반드시 Non-Blocking + EAGAIN 처리 조합 필요

Level Trigger 모드 (기본):
  데이터가 있는 동안 계속 통지
  → Blocking read()도 사용 가능하지만 Non-Blocking 권장
```

---

## 💻 실전 실험

### 실험 1: EAGAIN 동작 직접 확인

```c
/* eagain_demo.c */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <sys/socket.h>

int main() {
    /* 파이프로 Non-Blocking 테스트 */
    int pipefd[2];
    pipe(pipefd);

    /* 읽기 쪽 Non-Blocking 설정 */
    int flags = fcntl(pipefd[0], F_GETFL, 0);
    fcntl(pipefd[0], F_SETFL, flags | O_NONBLOCK);

    char buf[1024];

    /* 데이터 없을 때 read() */
    ssize_t n = read(pipefd[0], buf, sizeof(buf));
    if (n < 0 && errno == EAGAIN) {
        printf("EAGAIN 반환 (데이터 없음) - 정상!\n");
    }

    /* 데이터 쓰기 */
    write(pipefd[1], "Hello", 5);

    /* 데이터 있을 때 read() */
    n = read(pipefd[0], buf, sizeof(buf));
    printf("읽기 성공: %.*s (%zd bytes)\n", (int)n, buf, n);

    /* 다시 빈 상태 */
    n = read(pipefd[0], buf, sizeof(buf));
    printf("다시 EAGAIN: %s\n", (n < 0 && errno == EAGAIN) ? "yes" : "no");

    close(pipefd[0]);
    close(pipefd[1]);
    return 0;
}
```

```bash
$ gcc eagain_demo.c -o eagain_demo && ./eagain_demo
EAGAIN 반환 (데이터 없음) - 정상!
읽기 성공: Hello (5 bytes)
다시 EAGAIN: yes
```

### 실험 2: strace로 EAGAIN 발생 빈도 측정

```bash
# busy-wait 시나리오 시뮬레이션
$ strace -c -e trace=read ./busy_wait_program 2>&1 | grep read

# 이벤트 루프 기반 프로그램
$ strace -c -e trace=read,epoll_wait ./event_loop_program 2>&1

# 비교 포인트:
# busy-wait: read 호출 수 >> 실제 데이터 읽기 수
# 이벤트 루프: read 호출 수 ≈ 실제 데이터 읽기 수 (EAGAIN 거의 없음)

# Redis의 실제 시스템 콜 패턴 확인
$ strace -c -p $(pgrep redis-server) sleep 5 2>&1 | head -15
# epoll_wait가 대부분, read는 실제 데이터 있을 때만
```

### 실험 3: Non-Blocking write()의 EAGAIN

```bash
# 소켓 송신 버퍼를 채워서 write() EAGAIN 유발
$ cat << 'EOF' > /tmp/write_eagain.py
import socket, time, fcntl, os, errno

# 서버 연결 (nc 등으로 미리 대기)
s = socket.socket()
s.connect(('127.0.0.1', 9999))

# Non-Blocking 설정
fcntl.fcntl(s, fcntl.F_SETFL, os.O_NONBLOCK)

data = b'X' * 65536  # 64KB
total = 0
while True:
    try:
        n = s.send(data)
        total += n
        print(f"보낸 바이트: {total}")
    except BlockingIOError:  # EAGAIN
        print(f"EAGAIN - 송신 버퍼 가득 참 (총 {total}bytes)")
        time.sleep(0.1)
EOF

# 수신 안 하는 서버 실행 후 테스트
$ nc -l 9999 > /dev/null &  # 수신만 하고 버리는 서버
$ python3 /tmp/write_eagain.py
# 일정 바이트 후 EAGAIN 발생 = 소켓 송신 버퍼 크기
```

---

## 📊 성능/비용 비교

| 방식 | CPU 사용 (대기 중) | 레이턴시 | 구현 복잡도 |
|------|-----------------|---------|-----------|
| Blocking I/O | 0% (잠듦) | 낮음 (즉시 Wake-up) | 낮음 |
| Non-Blocking + busy-wait | ~100% (루프) | 매우 낮음 (폴링 주기) | 중간 |
| Non-Blocking + poll/epoll | ~0% (대기) | 낮음 | 중간~높음 |

**busy-wait vs epoll 시스템 콜 비교 (1000개 연결, 초당 100 이벤트)**:

```
busy-wait:
  초당 read() 호출: 1000 연결 * 폴링 빈도 = 수백만 회
  실제 데이터 있는 read(): 100회 (이벤트 수)
  낭비 비율: 99.99%+

epoll + Non-Blocking:
  초당 epoll_wait() 호출: ~100회 (이벤트마다)
  초당 read() 호출: ~100회 (이벤트 수만큼)
  낭비 없음: 거의 0%
```

---

## ⚖️ 트레이드오프

**Non-Blocking I/O의 복잡성**

Non-Blocking I/O는 partial read/write를 직접 처리해야 한다. 10KB를 write()했는데 3KB만 성공하면(나머지는 EAGAIN) 나머지 7KB를 나중에 다시 시도해야 한다. Blocking I/O는 이런 부분 처리가 자동이지만, Non-Blocking은 애플리케이션이 직접 상태를 관리해야 한다. Netty, libuv 같은 프레임워크가 이 복잡성을 추상화하는 이유다.

**busy-wait이 의미 있는 경우**

고빈도 트레이딩 시스템처럼 레이턴시가 마이크로초 단위로 중요한 경우, epoll의 wake-up 지연조차 너무 크다. 이때 의도적으로 busy-wait을 사용하고 CPU 코어 하나를 전용으로 할당한다. 커널 우회(DPDK 같은 방식)도 같은 원리다. 일반 서버에서는 절대 사용하면 안 된다.

---

## 📌 핵심 정리

```
O_NONBLOCK:
  read() 데이터 없음 → EAGAIN(-11) 즉시 반환 (블로킹 없음)
  write() 버퍼 가득 → EAGAIN 즉시 반환
  설정: fcntl(fd, F_SETFL, flags | O_NONBLOCK)

EAGAIN 처리:
  에러가 아님! 재시도 신호
  errno == EAGAIN → 이벤트 대기 후 재시도

busy-wait의 문제:
  EAGAIN → 즉시 재시도 → CPU 100% 소비
  해결: poll/epoll과 조합

올바른 Non-Blocking 패턴:
  1. fd를 O_NONBLOCK으로 설정
  2. epoll에 등록
  3. epoll_wait()으로 이벤트 대기 (CPU 양보)
  4. 이벤트 발생 → Non-Blocking read()
  5. EAGAIN까지 읽기 (ET 모드에서 필수)

핵심 진단:
  strace -c -e trace=read: EAGAIN 비율 확인
  EAGAIN 비율 높음 → epoll 없이 busy-wait 의심
  CPU 100% + I/O 없음 → busy-wait 확인
```

---

## 🤔 생각해볼 문제

**Q1.** Non-Blocking 소켓에서 `write(fd, buf, 10240)` 호출 시 4096만 성공하고 EAGAIN이 반환됐다. 나머지 6144바이트는 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

나머지 6144바이트를 버퍼에 보관하고, 해당 fd에 대해 `EPOLLOUT` 이벤트를 epoll에 등록한다. 소켓 송신 버퍼에 여유가 생기면 `EPOLLOUT` 이벤트가 발생하고, 그 시점에 남은 데이터를 다시 write()한다. 이 상태 관리(쓰기 버퍼 유지, EPOLLOUT 등록/해제)를 직접 구현하는 것이 Non-Blocking I/O의 복잡성이다. Netty는 이를 `ChannelOutboundBuffer`로 추상화한다.

</details>

**Q2.** `epoll` ET(Edge Trigger) 모드를 쓸 때 반드시 Non-Blocking I/O를 써야 하는 이유는?

<details>
<summary>해설 보기</summary>

ET 모드는 상태 변화(데이터 없음→있음)가 발생할 때만 1번 통지한다. 1번 통지 후 EAGAIN이 날 때까지 모든 데이터를 읽어야 한다. 만약 Blocking read()를 쓰면, 버퍼의 마지막 데이터를 읽은 후 다음 read()가 무한 블로킹에 빠진다(더 이상 데이터가 없으므로). Non-Blocking이라면 EAGAIN을 받아 루프를 종료할 수 있다. ET 모드에서 Non-Blocking은 필수다.

</details>

**Q3.** Redis는 단일 스레드 이벤트 루프를 쓴다. 클라이언트 연결 수가 1만 개여도 Non-Blocking + epoll로 처리할 수 있는 이유는?

<details>
<summary>해설 보기</summary>

epoll이 관심 있는 fd를 커널 내부 자료구조(레드-블랙 트리)로 관리하기 때문이다. 1만 개의 연결 중 실제로 데이터가 온 연결만 `epoll_wait()`이 반환한다. Redis는 반환된 이벤트 목록을 순서대로 처리한다. 각 명령어 처리가 마이크로초 단위로 매우 빠르기 때문에 단일 스레드로도 충분하다. 만약 하나의 명령어가 수 ms를 소비하면(예: 키 수백만 개의 `KEYS *`) 다른 클라이언트가 대기하는 레이턴시 스파이크가 발생한다.

</details>

---

**[⬅️ 이전: Blocking I/O](./02-blocking-io.md)** | **[홈으로 🏠](../README.md)** | **[다음: I/O Multiplexing ➡️](./04-io-multiplexing-select-poll.md)**
