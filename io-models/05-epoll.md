# 05. epoll — 이벤트 기반 O(1) 처리

## 🎯 핵심 질문

- `epoll_create()`, `epoll_ctl()`, `epoll_wait()` 세 단계는 각각 무엇을 하는가?
- 커널이 관심 fd 목록을 "기억"하는 방식이 select/poll과 어떻게 다른가?
- 레벨 트리거(LT)와 엣지 트리거(ET)의 차이는 무엇이고, 각각 언제 쓰는가?
- Redis 이벤트 루프가 epoll을 어떻게 활용하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Nginx가 수만 개의 동시 연결을 처리하면서도 CPU를 적게 쓰는 이유는 epoll 덕분이다.** 이벤트가 발생한 연결만 처리하고, 대기 중인 수만 개의 연결을 스캔하지 않는다. epoll의 O(1) 이벤트 감지가 이것을 가능하게 한다.

**Netty의 `EpollEventLoop`이 `NioEventLoop`보다 성능이 좋은 이유는?** Java NIO `Selector`도 내부적으로 epoll을 쓰지만, Netty의 네이티브 Epoll 트랜스포트는 JNI를 통해 epoll을 직접 호출해 JDK 래핑 오버헤드를 줄인다. 또한 ET(Edge Trigger) 모드를 활용해 이벤트 처리 효율을 높인다.

**Redis의 `ae_epoll.c`를 읽으면 epoll의 실제 사용 패턴이 보인다.** Redis 소스코드는 epoll API를 가장 깔끔하게 사용하는 예시 중 하나다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```c
// 실수 1: ET 모드에서 EAGAIN까지 읽지 않음
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);  // EPOLLIN | EPOLLET

// epoll_wait 후:
int n = read(fd, buf, 1024);  // 부분 읽기
// 다시 epoll_wait → 이벤트 통지 안 옴!
// → 나머지 데이터가 버퍼에 있지만 ET는 재통지 안 함

// 올바른 ET 처리: EAGAIN까지 루프
while (1) {
    n = read(fd, buf, sizeof(buf));
    if (n == -1 && errno == EAGAIN) break;  // 다 읽었음
    if (n <= 0) { close_connection(fd); break; }
    process(buf, n);
}

// 실수 2: LT 모드에서 읽지 않으면 이벤트 반복
// LT 모드: 데이터가 있는 한 계속 POLLIN 통지
// 데이터를 읽지 않으면 epoll_wait마다 같은 fd가 반환됨
// → 무한 루프처럼 동작
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```c
/* epoll 이벤트 루프 기본 패턴 */
#include <sys/epoll.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>

#define MAX_EVENTS 64

int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

void run_event_loop(int listen_fd) {
    /* 1. epoll 인스턴스 생성 */
    int epfd = epoll_create1(EPOLL_CLOEXEC);

    /* 2. 리스닝 소켓 등록 (LT 모드, EPOLLIN) */
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = listen_fd };
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    struct epoll_event events[MAX_EVENTS];

    while (1) {
        /* 3. 이벤트 대기 (무한 블로킹) */
        int nready = epoll_wait(epfd, events, MAX_EVENTS, -1);

        for (int i = 0; i < nready; i++) {
            int fd = events[i].data.fd;

            if (fd == listen_fd) {
                /* 새 연결 */
                int conn = accept(listen_fd, NULL, NULL);
                set_nonblocking(conn);
                /* ET 모드로 등록 */
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = conn;
                epoll_ctl(epfd, EPOLL_CTL_ADD, conn, &ev);
            } else {
                /* 데이터 수신: ET이므로 EAGAIN까지 읽기 */
                char buf[4096];
                while (1) {
                    int n = read(fd, buf, sizeof(buf));
                    if (n == -1 && errno == EAGAIN) break;
                    if (n <= 0) {
                        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                        close(fd);
                        break;
                    }
                    process(buf, n);
                }
            }
        }
    }
}
```

---

## 🔬 내부 동작 원리

### epoll의 세 API

```
epoll_create1(flags):
  → 커널에 epoll 인스턴스 생성
  → 반환: epoll 인스턴스를 가리키는 fd (epfd)
  → 내부: 레드-블랙 트리 + 준비 목록 초기화

epoll_ctl(epfd, op, fd, event):
  op:
    EPOLL_CTL_ADD: fd를 감시 목록에 추가
    EPOLL_CTL_MOD: 이벤트 변경
    EPOLL_CTL_DEL: fd를 감시 목록에서 제거
  → 레드-블랙 트리에서 O(log N) 삽입/삭제
  → fd에 콜백 함수 등록 (이벤트 발생 시 호출됨)

epoll_wait(epfd, events, maxevents, timeout):
  → 준비 목록 확인 (비어있으면 블로킹)
  → 이벤트 발생 시 준비 목록에서 events 배열로 복사
  → 반환: 준비된 이벤트 수 (O(준비된 수))
```

### 커널 내부 자료구조

```
epoll 인스턴스 내부:

  [레드-블랙 트리] ← epoll_ctl()로 관리
  │  fd 1000 ──→ [epitem: events=EPOLLIN, callback=...]
  │  fd 1001 ──→ [epitem: events=EPOLLIN|EPOLLET, callback=...]
  │  fd 1002 ──→ [epitem: events=EPOLLOUT, callback=...]
  │  ... (N개 fd, O(log N) 삽입/삭제)
  │
  [준비 목록 (이중 연결 리스트)] ← 이벤트 발생 시 추가
  │  [fd 1001: EPOLLIN 발생]
  │  [fd 1003: EPOLLIN 발생]
  └→ epoll_wait()이 이것을 반환

이벤트 발생 처리:
  NIC 인터럽트 → TCP 소켓 수신 버퍼에 데이터 추가
  → 소켓의 epoll 콜백 호출
  → 해당 fd의 epitem을 준비 목록에 추가
  → epoll_wait()이 블로킹 중이면 Wake-up

select/poll과의 핵심 차이:
  select/poll: "어떤 fd가 준비됐는지 지금 검사해줘" (매번 O(N))
  epoll:       "이벤트 발생하면 알려줘" (콜백으로 O(1) 등록)
```

### 레벨 트리거(LT) vs 엣지 트리거(ET)

```
Level Trigger (LT, 기본값):
  "소켓 수신 버퍼에 데이터가 있는 동안" 계속 통지

  타임라인:
    t=0: 100바이트 도착
    t=1: epoll_wait() → fd 반환 (데이터 있음)
    t=2: read(50바이트)  ← 50바이트만 읽음
    t=3: epoll_wait() → 또 fd 반환! (아직 50바이트 남음)
    t=4: read(50바이트)  ← 나머지 읽음
    t=5: epoll_wait() → fd 반환 안 함 (비었음)

  장점: 구현이 안전 (데이터를 놓치지 않음)
  단점: 데이터가 남은 동안 반복 통지 (오버헤드)
  적합: Blocking fd와 조합, 구현 단순함 원할 때

Edge Trigger (ET, EPOLLET 플래그):
  "소켓 수신 버퍼에 데이터가 새로 추가됐을 때" 1번만 통지

  타임라인:
    t=0: 100바이트 도착
    t=1: epoll_wait() → fd 반환 (변화 감지!)
    t=2: read(50바이트)만 읽음
    t=3: epoll_wait() → fd 반환 안 함! (변화 없음)
    t=4: 데이터 50바이트가 버퍼에 있지만 통지 없음!
    ↑ 버그! → EAGAIN까지 모두 읽어야 함

  올바른 ET 처리:
    t=1: epoll_wait() → fd 반환
    t=2: loop { read(buf) until EAGAIN }  ← 전부 읽기
    t=3: epoll_wait() → (비어있으므로) 대기

  장점: 이벤트 최소화, 고성능 서버에서 활용
  단점: 반드시 Non-Blocking + EAGAIN 처리 필수, 구현 복잡
  적합: Netty ET 모드, 고성능 서버, 숙련된 구현

EPOLLONESHOT:
  이벤트 발생 후 자동으로 감시 목록에서 비활성화
  멀티스레드에서 같은 fd를 여러 스레드가 처리하는 것 방지
  처리 완료 후 EPOLL_CTL_MOD으로 재활성화
```

### Redis ae_epoll.c 분석

```c
/* Redis 이벤트 루프 핵심 (ae_epoll.c 단순화) */

/* epoll 인스턴스 생성 */
state->epfd = epoll_create(1024);

/* fd 등록 (aeApiAddEvent) */
struct epoll_event ee = {0};
ee.events = mask & AE_READABLE ? EPOLLIN : 0;
ee.events |= mask & AE_WRITABLE ? EPOLLOUT : 0;
ee.data.fd = fd;
epoll_ctl(state->epfd, op, fd, &ee);

/* 이벤트 대기 (aeApiPoll) - Redis 메인 루프의 핵심 */
int retval = epoll_wait(state->epfd,
                        state->events,
                        eventLoop->setsize,
                        tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);

/* 반환된 이벤트만 처리 */
for (j = 0; j < numevents; j++) {
    struct epoll_event *e = state->events + j;
    int fd = e->data.fd;
    int mask = 0;
    if (e->events & EPOLLIN)  mask |= AE_READABLE;
    if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
    eventLoop->fired[j].fd   = fd;
    eventLoop->fired[j].mask = mask;
}
```

```
Redis 이벤트 루프 전체 흐름:

while (1) {
    1. beforeSleep() - 대기 전 정리 작업
    2. epoll_wait() - 이벤트 대기 (유휴 시 CPU 사용 0%)
    3. 발생한 이벤트 처리 (EPOLLIN: 명령어 읽기, EPOLLOUT: 응답 쓰기)
    4. 타이머 이벤트 처리 (만료 키 삭제, 통계 업데이트 등)
}

Redis가 단일 스레드로 빠른 이유:
  - 대부분의 시간: epoll_wait()에서 대기 (CPU 0%)
  - 이벤트 발생 시: 마이크로초 단위로 명령어 처리
  - 컨텍스트 스위칭 없음, TLB/캐시 효율 극대화
```

---

## 💻 실전 실험

### 실험 1: epoll 동작 단계별 확인

```bash
# strace로 epoll API 호출 순서 확인
$ strace -e trace=epoll_create1,epoll_ctl,epoll_wait -p $(pgrep nginx) 2>&1 | head -30

# 출력 예시:
# epoll_create1(EPOLL_CLOEXEC) = 7                    ← 인스턴스 생성
# epoll_ctl(7, EPOLL_CTL_ADD, 6, {EPOLLIN, {fd=6}}) = 0  ← fd 등록
# epoll_wait(7, [{EPOLLIN, {fd=6}}], 512, -1) = 1    ← 이벤트 대기 → 1개 반환

# Redis의 epoll 사용 패턴
$ strace -c -e trace=epoll_wait -p $(pgrep redis-server) sleep 5
# epoll_wait 호출 중 대기 시간 비율 높음 = 건강한 이벤트 루프
```

### 실험 2: LT vs ET 동작 차이 실험

```c
/* lt_vs_et.c - LT와 ET의 동작 차이 시각화 */
#include <sys/epoll.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void test_trigger_mode(const char *mode, int use_et) {
    int pipefd[2];
    pipe(pipefd);

    int epfd = epoll_create1(0);
    struct epoll_event ev = {
        .events = EPOLLIN | (use_et ? EPOLLET : 0),
        .data.fd = pipefd[0]
    };
    epoll_ctl(epfd, EPOLL_CTL_ADD, pipefd[0], &ev);

    /* 100바이트 쓰기 */
    write(pipefd[1], "A", 50);  // 50바이트만 쓰기

    struct epoll_event events[10];
    int nready;

    printf("\n=== %s 모드 ===\n", mode);

    /* 1차 epoll_wait - 이벤트 있음 */
    nready = epoll_wait(epfd, events, 10, 100);
    printf("1차 epoll_wait: %d개 이벤트\n", nready);

    /* 30바이트만 읽기 (20바이트 남김) */
    char buf[30];
    read(pipefd[0], buf, 30);
    printf("30바이트 읽음 (20바이트 남음)\n");

    /* 2차 epoll_wait */
    nready = epoll_wait(epfd, events, 10, 100);
    printf("2차 epoll_wait: %d개 이벤트 (LT=1, ET=0 예상)\n", nready);

    /* 추가 데이터 쓰기 */
    write(pipefd[1], "B", 10);
    nready = epoll_wait(epfd, events, 10, 100);
    printf("추가 데이터 후 epoll_wait: %d개 이벤트\n", nready);

    close(pipefd[0]); close(pipefd[1]); close(epfd);
}

int main() {
    test_trigger_mode("Level Trigger", 0);
    test_trigger_mode("Edge Trigger",  1);
    return 0;
}
```

```bash
$ gcc lt_vs_et.c -o lt_vs_et && ./lt_vs_et
=== Level Trigger 모드 ===
1차 epoll_wait: 1개 이벤트
30바이트 읽음 (20바이트 남음)
2차 epoll_wait: 1개 이벤트  ← LT: 남은 데이터 있어서 재통지
추가 데이터 후 epoll_wait: 1개 이벤트

=== Edge Trigger 모드 ===
1차 epoll_wait: 1개 이벤트
30바이트 읽음 (20바이트 남음)
2차 epoll_wait: 0개 이벤트  ← ET: 상태 변화 없음, 재통지 없음!
추가 데이터 후 epoll_wait: 1개 이벤트  ← 새 데이터 추가 = 변화 감지
```

### 실험 3: epoll 성능 측정

```bash
# 연결 1000개 서버에서 epoll vs poll 처리량 비교
$ git clone https://github.com/nicowillis/benchmarks  # 또는 직접 구현
$ ./epoll_bench 1000 &  # 1000 연결 epoll 서버
$ ./poll_bench 1000 &   # 1000 연결 poll 서버

# 각 서버에 동일 부하
$ ab -n 100000 -c 1000 http://localhost:8001/  # epoll
$ ab -n 100000 -c 1000 http://localhost:8002/  # poll

# CPU 사용률 비교
$ top -p $(pgrep epoll_bench),$(pgrep poll_bench)
# epoll: CPU ~10%  / poll: CPU ~35% (연결 수 클수록 차이 증가)
```

---

## 📊 성능/비용 비교

| 작업 | select/poll | epoll |
|------|-----------|-------|
| fd 등록 | 매번 복사 | O(log N), 최초 1회 |
| 이벤트 감지 | O(N) 스캔 | O(1) 콜백 |
| 결과 반환 | O(N) 스캔 | O(이벤트 수) |
| fd 수 제한 | 1024 (select) | 시스템 한도 |
| 메모리 | O(N) 복사 | O(N) 트리 (커널 유지) |

**epoll이 select/poll 대비 우위를 보이는 구간**:

```
연결 100개 이하: select/poll과 성능 차이 거의 없음
연결 1,000개:   epoll이 약 2~5배 빠름
연결 10,000개:  epoll이 약 20~50배 빠름
연결 100,000개: epoll만 실용적 (select/poll은 사실상 불가)
```

---

## ⚖️ 트레이드오프

**LT vs ET 선택**

LT가 구현이 쉽고 안전하다. 데이터를 일부만 읽어도 다음 `epoll_wait()`에서 다시 통지받으므로 데이터 손실이 없다. ET는 구현이 복잡하지만 이벤트 통지 횟수가 줄어 `epoll_wait()` 호출 오버헤드가 감소한다. Nginx는 ET를 사용해 처리량을 높인다. 처음 구현 시에는 LT, 성능 튜닝이 필요할 때 ET로 전환하는 방식이 일반적이다.

**epoll의 경우도 fd 수가 많으면 메모리가 필요하다**

레드-블랙 트리는 각 fd마다 `epitem` 구조체를 유지한다. 수백만 개의 fd를 등록하면 커널 메모리가 수 GB에 달할 수 있다. 실제로 이 수준의 연결을 다루는 경우는 드물지만, 매우 많은 연결을 처리하는 시스템에서는 커널 메모리 사용량도 고려해야 한다.

---

## 📌 핵심 정리

```
epoll 세 단계:
  epoll_create1() → epoll 인스턴스(fd) 생성
  epoll_ctl()     → fd 등록/수정/삭제 (레드-블랙 트리, O(log N))
  epoll_wait()    → 이벤트 발생 대기 → 준비 목록 반환 (O(이벤트 수))

select/poll과의 핵심 차이:
  "매번 전달 + 전체 스캔" vs "한 번 등록 + 콜백으로 통지"

LT (Level Trigger, 기본):
  데이터가 있는 동안 계속 통지
  안전, 구현 쉬움, Blocking fd와 조합 가능

ET (Edge Trigger, EPOLLET):
  상태 변화 시 1번만 통지
  반드시 Non-Blocking + EAGAIN까지 읽기
  고성능 서버에서 활용 (Nginx, Netty)

Redis 이벤트 루프:
  epoll_wait() 대기 → 이벤트 발생 → 명령어 처리 (마이크로초)
  단일 스레드 + epoll = 컨텍스트 스위칭 없음 + O(1) 이벤트 처리

Java NIO:
  Selector.select() = 내부적으로 epoll_wait()
  SelectionKey = epoll_ctl() 등록과 동일
```

---

## 🤔 생각해볼 문제

**Q1.** `epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev)` 호출 시 `ev.data.fd = fd`를 설정한다. `ev.data`를 `fd` 대신 포인터로 설정하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

`epoll_event.data`는 유니온이어서 `fd`, `u32`, `u64`, `ptr` 중 하나를 저장할 수 있다. 포인터를 저장하면 `epoll_wait()` 반환 시 fd 번호가 아닌 커넥션 구조체 포인터를 바로 얻을 수 있다. fd를 저장하면 `epoll_wait()` 후 fd 번호로 해시맵에서 구조체를 찾아야 하는 추가 조회가 필요하다. 포인터를 직접 저장하면 조회 없이 O(1)로 해당 연결의 상태를 접근할 수 있다. Nginx와 많은 이벤트 루프 구현이 이 방식을 사용한다.

</details>

**Q2.** 멀티스레드 서버에서 여러 스레드가 같은 `epfd`로 `epoll_wait()`을 호출해도 되는가?

<details>
<summary>해설 보기</summary>

**가능하지만 주의가 필요하다.** 여러 스레드가 같은 epfd로 대기하면 이벤트 발생 시 하나의 스레드만 깨어난다(thundering herd 방지를 위한 커널 설계). 이를 활용해 Accept 소켓 하나에 여러 워커 스레드가 `epoll_wait()`을 하는 패턴이 가능하다. 다만 같은 fd의 이벤트를 여러 스레드가 처리하면 경합이 발생한다. `EPOLLONESHOT` 플래그로 한 번 처리 후 자동 비활성화해 경합을 방지할 수 있다.

</details>

**Q3.** Redis가 ET 대신 LT를 사용하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

Redis는 LT(Level Trigger) 모드를 기본으로 사용한다. 이유는 단일 스레드 이벤트 루프에서 구현의 단순성을 유지하기 위해서다. ET 모드에서는 EAGAIN까지 모든 데이터를 읽어야 하는데, Redis가 단일 명령어를 처리하는 동안 다른 클라이언트가 대기해야 한다. 큰 값을 읽는 동안 이벤트 루프가 차단되지 않도록 LT 모드에서 부분적으로 읽고 나머지는 다음 이벤트 루프 사이클에서 처리하는 방식이 더 적합하다. Redis 6.0의 Threaded I/O에서도 이 원칙은 유지된다.

</details>

---

**[⬅️ 이전: I/O Multiplexing](./04-io-multiplexing-select-poll.md)** | **[홈으로 🏠](../README.md)** | **[다음: I/O 모델이 백엔드에 미치는 영향 ➡️](./06-io-model-backend-impact.md)**
