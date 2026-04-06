# 04. I/O Multiplexing — select/poll의 한계

## 🎯 핵심 질문

- `select()`는 어떻게 하나의 스레드로 여러 fd를 감시하는가?
- 왜 `select()`는 fd가 많아질수록 느려지는가?
- `poll()`은 `select()`의 무엇을 개선했고, 어떤 한계가 남았는가?
- C10K 문제는 정확히 어느 지점에서 발생하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**레거시 코드에서 `select()` 기반 서버가 연결 1024개 이상에서 불안정한 이유를 설명해야 한다.** `select()`는 `fd_set` 비트마스크를 사용하고 `FD_SETSIZE=1024`가 컴파일 타임에 고정된다. 이 제한을 넘으면 정의되지 않은 동작이 발생한다.

**`poll()` 기반 코드가 동시 연결 수만 개에서 왜 CPU를 많이 소비하는지 이해해야 한다.** `poll()`은 매번 전체 fd 배열을 커널에 복사하고, 커널이 O(N) 스캔을 수행한다. 연결 1만 개에서 이벤트 하나 찾으려면 1만 개 fd를 모두 검사한다.

**Nginx가 `select()` 대신 `epoll`을 사용하도록 컴파일해야 하는 이유를 알아야 한다.** 운영 환경에서 `--with-select_module` vs `--with-epoll_module` 선택의 근거가 여기에 있다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```c
// 실수 1: FD_SETSIZE 초과 (정의되지 않은 동작)
fd_set read_fds;
FD_ZERO(&read_fds);
for (int i = 0; i < 2000; i++) {
    FD_SET(client_fds[i], &read_fds);  // fd > 1023이면 버퍼 오버플로!
}
select(max_fd + 1, &read_fds, NULL, NULL, NULL);

// 실수 2: select() 반환 후 어느 fd가 준비됐는지 O(N) 스캔
int nready = select(max_fd + 1, &read_fds, NULL, NULL, NULL);
for (int fd = 0; fd <= max_fd; fd++) {
    if (FD_ISSET(fd, &read_fds)) {
        handle(fd);  // 1024개 전체 스캔, 이벤트 1개여도
    }
}
```

```python
# 실수 3: Python에서 poll() 없이 select() 사용 (연결 많을 때)
import select
# select() 대신 selectors 모듈 사용 권장 (내부적으로 최적 방식 선택)
import selectors
sel = selectors.DefaultSelector()  # 리눅스에서 epoll 자동 선택
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```c
/* poll() 올바른 사용 (select보다 낫지만 epoll이 더 좋음) */
#include <poll.h>
#include <stdlib.h>

struct pollfd *fds = NULL;
int nfds = 0;
int capacity = 0;

void add_fd(int fd) {
    if (nfds == capacity) {
        capacity = capacity ? capacity * 2 : 16;
        fds = realloc(fds, capacity * sizeof(struct pollfd));
    }
    fds[nfds++] = (struct pollfd){ .fd = fd, .events = POLLIN };
}

void event_loop() {
    while (1) {
        int ready = poll(fds, nfds, -1);  // 무한 대기
        if (ready < 0) break;

        for (int i = 0; i < nfds && ready > 0; i++) {
            if (fds[i].revents & POLLIN) {
                handle_read(fds[i].fd);
                ready--;
            }
        }
        // 문제: nfds가 크면 이 루프가 O(N)
        // → epoll은 ready 목록만 반환하여 O(ready) 처리 가능
    }
}
```

---

## 🔬 내부 동작 원리

### select()의 동작 방식

```
select(nfds, readfds, writefds, exceptfds, timeout) 호출:

인자:
  nfds:     감시할 fd 최대값 + 1
  readfds:  읽기 이벤트를 감시할 fd 비트마스크
  writefds: 쓰기 이벤트를 감시할 fd 비트마스크
  timeout:  NULL=무한대기, 0=즉시반환, {sec,usec}=타임아웃

fd_set 비트마스크 구조:
  fd_set = 1024비트 배열 (= 128바이트)
  fd 번호 = 비트 위치
  FD_SET(3, &fds) → 비트 3을 1로 설정
  FD_ISSET(3, &fds) → 비트 3이 1인지 확인

커널 내부 동작:
  1. 유저 공간 fd_set을 커널로 복사 (128바이트)
  2. 0부터 nfds-1까지 모든 fd를 순회:
     for (fd = 0; fd < nfds; fd++) {
         if (FD_ISSET(fd, readfds) && fd_has_data(fd))
             fd_set 비트 설정
     }
  3. 결과 fd_set을 유저 공간으로 복사
  4. 준비된 fd 수 반환

문제점:
  ① nfds가 클수록 O(nfds) 스캔 → 1024개면 항상 1024번 검사
  ② 매번 fd_set을 커널에 복사 → 시스템 콜마다 128바이트
  ③ FD_SETSIZE = 1024 하드코딩 → fd > 1023 불가 (버퍼 오버플로)
  ④ fd_set이 select() 후 수정됨 → 매번 재초기화 필요
```

### poll()의 개선과 남은 한계

```
poll(fds, nfds, timeout) 호출:

struct pollfd {
    int   fd;      ← 감시할 fd
    short events;  ← 관심 이벤트 (POLLIN, POLLOUT 등)
    short revents; ← 발생한 이벤트 (커널이 채워줌)
};

select() 대비 개선:
  ① fd 수 제한 없음 (FD_SETSIZE 제약 없음)
  ② 구조체 배열로 fd, 이벤트, 결과를 명확히 분리
  ③ revents를 통해 어떤 이벤트가 발생했는지 명시

여전한 한계:
  ① 매번 전체 fds 배열을 커널로 복사
     → nfds * sizeof(struct pollfd) = nfds * 8바이트
     → 1만 fd = 80KB 복사 (시스템 콜마다!)
  ② 커널이 여전히 O(N) 스캔
     → 이벤트 1개여도 1만 fd 모두 검사
  ③ 반환 후 어느 fd가 준비됐는지 O(N) 스캔 필요
     → 애플리케이션도 O(N) 루프 불가피

성능 문제 시나리오 (1만 연결, 초당 100 이벤트):
  poll() 호출: 100회/초
  커널 복사: 100 * 80KB = 8MB/초 (불필요한 메모리 대역폭)
  커널 스캔: 100 * 10,000 = 1,000,000 fd 검사/초
  실제 이벤트: 100개/초
  낭비: 999,900 / 1,000,000 = 99.99%
```

### C10K 문제의 본질

```
1999년 Dan Kegel "The C10K Problem":
  "하나의 서버에서 동시 연결 10,000개를 처리할 수 없다"

당시 제약:
  1. 스레드 모델: 10,000 스레드 = 80GB 스택 메모리 (불가능)
  2. select(): FD_SETSIZE = 1024 (1024개 제한)
  3. poll(): O(N) 스캔 (10,000개 연결 * 스캔 = 성능 문제)

select/poll의 근본 문제:
  "관심 fd 목록"을 매번 커널에 전달해야 함
  → 커널이 목록을 "기억"하지 않음
  → 매번 새로 전달 + 전체 스캔

epoll의 해결:
  "관심 fd 목록"을 커널이 유지 (등록 후 지속)
  → 이벤트 발생한 fd만 반환 (O(1))
  → 전체 스캔 없음

실제 성능 비교 (10,000 연결):
  select():  ~100,000회 fd 검사/초 (나쁨)
  poll():    ~100,000회 fd 검사/초 (비슷)
  epoll():   ~이벤트 수 처리/초 (O(1))
```

### select/poll이 여전히 쓰이는 경우

```
select()가 적합한 경우:
  - 이식성이 중요한 경우 (POSIX 표준)
  - fd 수가 적을 때 (< 100)
  - 타임아웃 정밀도가 중요할 때 (마이크로초 단위)

poll()이 select()보다 나은 경우:
  - fd > 1024가 필요한 경우
  - fd_set 재초기화 없이 구조체 재사용
  - 이벤트 종류가 다양한 경우 (POLLERR, POLLHUP 등)

현대 코드에서의 선택:
  리눅스: epoll (압도적 우위)
  macOS/BSD: kqueue
  크로스 플랫폼: libevent, libuv (내부적으로 최적 방식 선택)
```

---

## 💻 실전 실험

### 실험 1: select() FD_SETSIZE 한계 확인

```c
/* select_limit.c */
#include <stdio.h>
#include <sys/select.h>

int main() {
    printf("FD_SETSIZE = %d\n", FD_SETSIZE);

    fd_set fds;
    FD_ZERO(&fds);

    // fd 번호 1023: 안전
    FD_SET(1023, &fds);
    printf("FD_SET(1023): 성공\n");

    // fd 번호 1024: 정의되지 않은 동작 (버퍼 오버플로!)
    // FD_SET(1024, &fds);  // ← 실행하면 안 됨!
    printf("FD_SET(1024): 버퍼 오버플로 위험!\n");

    // select()의 최대 감시 가능 fd 수
    printf("select() 최대 감시 fd: 0~%d\n", FD_SETSIZE - 1);
    return 0;
}
```

```bash
$ gcc select_limit.c -o select_limit && ./select_limit
FD_SETSIZE = 1024
FD_SET(1023): 성공
FD_SET(1024): 버퍼 오버플로 위험!
select() 최대 감시 fd: 0~1023
```

### 실험 2: poll() vs epoll 성능 비교 (연결 수 증가)

```bash
# 연결 수별 CPU 사용률 비교 도구 설치
$ apt-get install -y libevent-dev

# 부하 생성 + 성능 측정
# poll 기반 서버: 실행 후 CPU top 확인
$ top -p $(pgrep poll_server)

# epoll 기반 서버: 동일 부하에서 CPU top 확인
$ top -p $(pgrep epoll_server)

# perf stat으로 시스템 콜 횟수 비교
$ perf stat -e syscalls:sys_enter_poll \
  -p $(pgrep poll_server) sleep 10

$ perf stat -e syscalls:sys_enter_epoll_wait \
  -p $(pgrep epoll_server) sleep 10
```

### 실험 3: strace로 poll() 오버헤드 확인

```bash
# poll() 호출 시 복사되는 데이터 크기 측정
$ strace -e trace=poll -s 256 -p $SERVER_PID 2>&1 | head -20
# poll([{fd=4, events=POLLIN}, {fd=5, events=POLLIN}, ...], 1000, -1) = 1
#      ↑ 1000개 fd 구조체가 매번 커널로 복사됨
#        = 1000 * 8바이트 = 8KB (매 poll() 호출마다)

# poll() 호출 빈도와 소요 시간
$ strace -c -e trace=poll -p $SERVER_PID sleep 10 2>&1
# calls: 호출 횟수 × 8KB = 총 복사 바이트
```

---

## 📊 성능/비용 비교

| 방식 | fd 제한 | 커널 전달 | 이벤트 감지 | 결과 확인 |
|------|--------|---------|-----------|---------|
| select() | 1024 | 매번 128B | O(N) 스캔 | O(N) 스캔 |
| poll() | 무제한 | 매번 N*8B | O(N) 스캔 | O(N) 스캔 |
| epoll() | 무제한 | 최초 등록만 | O(1) | O(이벤트 수) |

**연결 수에 따른 select/poll 성능 저하**:

```
select/poll의 복잡도: O(N) where N = 감시 fd 수

fd 수   poll() 호출당 비용 (CPU 사이클 추정)
   10:  ~1,000 사이클  (무시할 수준)
  100:  ~10,000 사이클
 1000:  ~100,000 사이클 (주목할 수준)
10000:  ~1,000,000 사이클 (심각)
100000: ~10,000,000 사이클 (불가능 수준)

epoll은 fd 수와 무관, 이벤트 수 O(1):
10000 fd, 100 이벤트 → ~10,000 사이클 (1000fd poll과 동일)
```

---

## ⚖️ 트레이드오프

**select/poll의 단순성 vs epoll의 복잡성**

`select()`와 `poll()`은 API가 단순하고 POSIX 표준이라 이식성이 좋다. 소수의 fd를 감시하는 경우 epoll을 쓸 필요가 없다. 하지만 동시 연결 수가 수백을 넘으면 epoll이 압도적으로 우위다. Nginx는 컴파일 옵션으로 방식을 선택하고, libevent/libuv는 실행 시점에 최적 방식을 자동 선택한다.

**poll()의 fd 배열 관리 복잡성**

poll()은 fd 배열을 직접 관리해야 한다. 연결이 종료되면 배열에서 제거하고, 새 연결이 오면 추가한다. 중간에 빈 슬롯이 생기면 defrag가 필요하다. epoll은 `epoll_ctl(EPOLL_CTL_DEL)`로 간단히 제거할 수 있어 관리가 더 깔끔하다.

---

## 📌 핵심 정리

```
select():
  동작: 유저→커널 fd_set 복사 → O(N) 스캔 → 결과 복사
  한계: FD_SETSIZE=1024, 매번 O(N), 매번 재초기화 필요

poll():
  동작: 유저→커널 pollfd 배열 복사 → O(N) 스캔 → 결과 저장
  개선: fd 수 무제한, 구조체로 이벤트 명시
  한계: 여전히 O(N) 스캔, 매번 N*8바이트 복사

공통 문제:
  커널이 관심 fd 목록을 "기억"하지 않음
  → 매번 전달 + 전체 스캔 = O(N) 오버헤드

C10K 문제:
  10,000 연결에서 select/poll의 O(N) 오버헤드가 성능 한계
  epoll (리눅스), kqueue (BSD)로 해결

현대 선택:
  리눅스 서버: epoll
  크로스 플랫폼 코드: libevent, libuv (자동 선택)
  Python: selectors.DefaultSelector()
  Java NIO: Selector (내부적으로 epoll)
```

---

## 🤔 생각해볼 문제

**Q1.** 서버에서 fd 번호가 1025인 소켓이 생겼다. `select()`로 이 소켓을 감시하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

**표준 `select()`로는 불가능하다.** `FD_SETSIZE=1024`이므로 fd 1024 이상은 `FD_SET()`으로 설정할 수 없고, 설정해도 버퍼 오버플로가 발생한다. 해결책은 `poll()`이나 `epoll()`로 전환하는 것이다. 꼭 `select()`를 써야 한다면 `dup2()`로 fd 번호를 낮게 재배정하거나, 불필요한 fd를 닫아 번호를 낮추는 우회 방법이 있지만 권장하지 않는다.

</details>

**Q2.** `poll()`으로 10,000개 fd를 감시하는데, 1초에 이벤트가 1개만 발생한다. 1초 동안 커널이 수행하는 fd 스캔 횟수는?

<details>
<summary>해설 보기</summary>

이벤트가 1개 발생하는 순간 `poll()`이 반환하고 즉시 다시 호출되므로, 이벤트가 없는 동안은 하나의 `poll()` 호출이 블로킹 대기한다. 커널 스캔은 `poll()` 반환 시점에 일어나므로 1초에 약 2번(진입 시 + 반환 직전)의 O(N) 스캔이 발생한다. 정확히는 커널이 이벤트 발생을 감지할 때 모든 fd를 한 번 순회하므로, 타임아웃이 없으면 이벤트 발생 당 1번의 10,000 fd 스캔이 발생한다. 타임아웃이 있으면 타임아웃마다 추가 스캔이 발생한다.

</details>

**Q3.** Java의 `java.nio.channels.Selector`는 내부적으로 무엇을 사용하는가?

<details>
<summary>해설 보기</summary>

JVM 구현과 운영체제에 따라 다르다. 리눅스(OpenJDK)에서는 기본적으로 **epoll**을 사용한다. macOS에서는 **kqueue**를 사용한다. 구형 JVM이나 특정 설정에서는 `poll()`을 사용하기도 한다. `sun.nio.ch.EPollSelectorImpl` 클래스가 epoll을 래핑한다. Java NIO가 "Non-Blocking I/O"를 제공할 수 있는 것도 내부적으로 OS의 epoll/kqueue를 활용하기 때문이다.

</details>

---

**[⬅️ 이전: Non-Blocking I/O](./03-nonblocking-polling.md)** | **[홈으로 🏠](../README.md)** | **[다음: epoll ➡️](./05-epoll.md)**
