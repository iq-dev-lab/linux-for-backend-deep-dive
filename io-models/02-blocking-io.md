# 02. Blocking I/O — 프로세스가 잠드는 원리

## 🎯 핵심 질문

- `read()` 호출 시 데이터가 없으면 프로세스는 정확히 어떤 상태가 되는가?
- 데이터가 도착했을 때 잠든 프로세스를 깨우는 메커니즘은?
- Blocking I/O 기반 스레드 풀 모델은 왜 C10K(동시 연결 1만)에서 무너지는가?
- Blocking I/O가 적합한 워크로드는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Spring MVC(Tomcat)와 Spring WebFlux(Netty)의 근본적인 차이는 Blocking I/O vs Non-Blocking I/O다.** 이것이 단순히 "스타일 차이"가 아니라 커널 레벨의 동작 방식이 완전히 다르다는 것을 알아야 언제 WebFlux로 전환해야 하는지 판단할 수 있다.

**스레드 풀을 200에서 400으로 늘렸더니 처리량이 오히려 줄었다.** CPU 바운드 작업이라면 CPU 코어 수 이상의 스레드는 컨텍스트 스위칭 오버헤드만 증가시킨다. I/O 바운드 작업이라면 스레드가 대부분 `TASK_INTERRUPTIBLE` 상태로 잠들어 있어 스레드 수를 늘려도 처리량이 I/O 속도에 제한된다.

**JDBC 드라이버가 Blocking I/O를 사용하는 한, WebFlux를 써도 효과가 없다.** `WebFlux`의 이벤트 루프 스레드가 JDBC 호출에서 블로킹되면 이벤트 루프 전체가 멈춘다. R2DBC(Reactive JDBC)나 스레드 풀 오프로딩이 필요한 이유가 여기에 있다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```java
// 실수 1: WebFlux에서 Blocking 코드 혼용
@GetMapping("/user/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return Mono.fromCallable(() -> {
        // ← 이 안에서 JDBC 호출 (Blocking!)
        return jdbcTemplate.queryForObject(...);
        // → 이벤트 루프 스레드가 DB 응답까지 블로킹
        // → 다른 요청 처리 불가
    });
    // Mono.fromCallable은 여전히 이벤트 루프에서 실행됨
}

// 올바른 접근: 별도 스레드 풀에서 실행
return Mono.fromCallable(() -> jdbcTemplate.queryForObject(...))
           .subscribeOn(Schedulers.boundedElastic());  // I/O 전용 스레드 풀
```

```bash
# 실수 2: 스레드 수만 늘려서 처리량 개선 시도
# application.properties:
server.tomcat.threads.max=1000  # 1000개로 늘림
# → 1000개 스레드가 모두 DB 응답 대기 (블로킹)
# → 실제로 CPU를 쓰는 스레드는 DB 응답 수에 제한됨
# → 1000개 스레드의 스택 메모리만 낭비 (1GB)
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 현재 스레드 상태 분포 확인
$ jstack $PID | grep "java.lang.Thread.State" | sort | uniq -c
  180 java.lang.Thread.State: WAITING (on object monitor)
   15 java.lang.Thread.State: RUNNABLE
    5 java.lang.Thread.State: TIMED_WAITING (sleeping)
# WAITING 180개 = 스레드 대부분이 I/O 또는 락 대기
# RUNNABLE 15개만 실제로 일하는 중

# OS 레벨에서 스레드 상태 확인
$ for tid in $(ls /proc/$PID/task/); do
    state=$(awk '/State:/{print $2}' /proc/$PID/task/$tid/status 2>/dev/null)
    echo "$state"
  done | sort | uniq -c
# S(sleeping) 180  ← TASK_INTERRUPTIBLE 상태 (I/O 대기)
# R(running)   15  ← 실행 중
```

---

## 🔬 내부 동작 원리

### Blocking read()의 전체 흐름

```
[애플리케이션 스레드]           [커널]                [NIC/디스크]

read(sockfd, buf, 4096)
       │
       ▼ syscall
  sys_read()
       │
  소켓 수신 버퍼 확인
       │
       └── 비어있음!
              │
              ▼
       프로세스 상태 변경:
       TASK_RUNNING → TASK_INTERRUPTIBLE
              │
              ▼
       소켓 대기 큐에 등록
       schedule() 호출 → 스케줄러가 다른 프로세스 선택
              │
              ▼
       [잠든 상태: CPU 사용 없음]
              │
              │                         패킷 도착!
              │                              │
              │                  NIC → DMA → sk_buff (수신 버퍼)
              │                              │
              │                  소켓 대기 큐의 프로세스 Wake-up
              │                  (wake_up_interruptible())
              │
              ▼
       TASK_INTERRUPTIBLE → TASK_RUNNABLE
       런큐에 추가 → 스케줄러가 CPU 할당
              │
              ▼
  sk_buff → copy_to_user → buf
  반환값: 읽은 바이트 수
```

### 인터럽트와 Wake-up 메커니즘

```
NIC가 패킷 수신 시:

1. NIC → DMA 컨트롤러: 패킷을 커널 메모리(sk_buff)에 직접 기록
2. DMA 완료 → NIC가 CPU에 인터럽트 신호 발생
3. CPU: 현재 실행 중이던 프로세스 일시 중단
4. 인터럽트 핸들러(IRQ) 진입
5. sk_buff를 TCP/IP 스택으로 전달
6. 올바른 소켓의 수신 큐에 sk_buff 추가
7. 해당 소켓에서 대기 중인 프로세스 Wake-up:
   wake_up_interruptible(&sock->wq->wait)
8. 인터럽트 핸들러 종료 → 이전 프로세스 재개
9. 스케줄러가 다음 타임슬라이스에 깨어난 프로세스 실행
10. copy_to_user()로 데이터 전달

중요: Wake-up은 즉시 실행이 아니다
  → 스케줄러 큐에 추가될 뿐
  → CPU가 해당 프로세스를 선택할 때까지 대기
```

### 스레드 1개 = 연결 1개의 한계

```
Blocking I/O 서버 (Apache 방식):

클라이언트 1 → [스레드 1: read() 블로킹 대기]
클라이언트 2 → [스레드 2: read() 블로킹 대기]
클라이언트 3 → [스레드 3: read() 블로킹 대기]
...
클라이언트 N → [스레드 N: read() 블로킹 대기]

모든 스레드가 I/O 대기 (TASK_INTERRUPTIBLE):
  → CPU는 거의 유휴 상태
  → 메모리: N * 스택 크기 (기본 8MB * N)
  → N = 10,000이면 스택만 80GB 필요!
  → 컨텍스트 스위칭: 패킷 도착마다 발생

문제점:
  1. 메모리 폭발: 연결 수 * 스택 크기
  2. 컨텍스트 스위칭 오버헤드: 스레드 수 * 전환 비용
  3. OS 스레드 생성 한도
  4. 스케줄러 부하: N개 스레드 관리

C10K 문제 (1999년 Dan Kegel):
  "동시 연결 10,000개를 처리할 수 없다"
  → 스레드 기반 서버의 구조적 한계 지적
  → Non-Blocking I/O + Multiplexing으로 해결
```

### Blocking I/O가 적합한 경우

```
Blocking I/O의 장점:
  ✓ 코드가 단순 (동기적 흐름, 콜백 없음)
  ✓ 스택 트레이스가 명확 (디버깅 쉬움)
  ✓ 각 연결이 독립적 (장애 격리)

적합한 워크로드:
  - 연결 수 << 스레드 풀 크기인 경우 (내부 마이크로서비스)
  - I/O 대기 시간이 짧은 경우 (로컬 DB, 캐시)
  - CPU 집약적 작업 (스레드가 실제로 계산 중)
  - JDBC 같은 Blocking API를 사용해야 하는 경우
  - 팀의 동기 코드 친숙도가 높은 경우

부적합한 워크로드:
  - 외부 API 호출 (수백 ms 대기)
  - 파일 스트리밍 (대용량, 긴 연결)
  - 동시 연결 수가 스레드 풀 크기를 크게 초과하는 경우
```

---

## 💻 실전 실험

### 실험 1: 프로세스가 잠드는 순간 관찰

```c
/* blocking_demo.c */
#include <stdio.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>

int main() {
    int server = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(9999),
        .sin_addr.s_addr = INADDR_ANY
    };
    int opt = 1;
    setsockopt(server, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    bind(server, (struct sockaddr*)&addr, sizeof(addr));
    listen(server, 1);

    printf("PID: %d - accept() 대기 중...\n", getpid());
    int client = accept(server, NULL, NULL);  // ← 여기서 블로킹

    char buf[1024];
    printf("연결됨. read() 대기 중...\n");
    int n = read(client, buf, sizeof(buf));   // ← 여기서 블로킹

    printf("수신: %.*s\n", n, buf);
    return 0;
}
```

```bash
$ gcc blocking_demo.c -o blocking_demo
$ ./blocking_demo &
PID: 12345

# 별도 터미널에서 프로세스 상태 관찰
$ cat /proc/12345/status | grep State
State: S (sleeping)   ← TASK_INTERRUPTIBLE, accept()에서 블로킹

# 연결 수립 후 read()에서 블로킹
$ telnet localhost 9999 &
$ cat /proc/12345/status | grep State
State: S (sleeping)   ← read()에서 블로킹

# 데이터 전송 → Wake-up
$ echo "hello" | nc localhost 9999
# State: R (running) → S (sleeping) 순간 전환
```

### 실험 2: Blocking I/O 서버 처리량 한계 측정

```bash
# Tomcat 기반 Spring Boot 부하 테스트
# application.properties: server.tomcat.threads.max=50

# 동시 요청 50개 (스레드 풀 한도):
$ ab -n 1000 -c 50 http://localhost:8080/api/slow
# Requests per second: 450/sec  ← 정상

# 동시 요청 200개 (스레드 풀 초과):
$ ab -n 1000 -c 200 http://localhost:8080/api/slow
# Requests per second: 450/sec  ← 처리량 동일 (대기 큐에서 대기)
# 응답 시간: 4배 증가

# 스레드 상태 확인
$ jstack $(pgrep java) | grep "State:" | sort | uniq -c
# 50개가 BLOCKED, 나머지는 WAITING (요청 대기 큐)
```

### 실험 3: 스레드 수 vs 처리량 관계

```bash
# 스레드 수별 처리량 측정 (I/O 바운드 작업 가정: 10ms DB 지연)
for threads in 10 50 100 200 500; do
    # Tomcat 스레드 수 설정 후 재시작
    echo "server.tomcat.threads.max=$threads" > application.properties
    # 부하 테스트
    result=$(ab -n 2000 -c $threads http://localhost:8080/api/db 2>&1 | \
             grep "Requests per second" | awk '{print $4}')
    echo "스레드 $threads개: $result req/s"
done

# 예상 결과 (DB 쿼리 10ms):
# 스레드  10개: ~1000 req/s  (10 / 0.01s)
# 스레드  50개: ~5000 req/s
# 스레드 100개: ~8000 req/s  (컨텍스트 스위칭 오버헤드 시작)
# 스레드 200개: ~7000 req/s  (오버헤드 > 이득)
# 스레드 500개: ~5000 req/s  (역효과)
```

---

## 📊 성능/비용 비교

| 항목 | Blocking I/O (스레드 풀) | Non-Blocking I/O (이벤트 루프) |
|------|----------------------|------------------------------|
| 동시 연결당 메모리 | ~1MB (스택) | ~수 KB (이벤트 구조체) |
| 코드 복잡도 | 낮음 (동기) | 높음 (비동기, 콜백) |
| 디버깅 용이성 | 높음 (스택 트레이스 명확) | 낮음 (콜백 체인 추적) |
| I/O 대기 시 CPU | 낭비 없음 (잠듦) | 낭비 없음 (다른 이벤트 처리) |
| 연결 10,000개 | 스레드 10,000개 (불가능) | 스레드 수십 개 (가능) |
| JDBC 호환 | 자연스러움 | 블로킹 오프로딩 필요 |

**스레드 수에 따른 I/O 바운드 처리량 곡선**:

```
처리량
  ↑
  │              ★ 최적점 ≈ I/O 대기시간에 따른 적정 스레드 수
  │         ____/
  │        /    \___________  ← 컨텍스트 스위칭 오버헤드
  │       /
  │      /
  │     /
  └──────────────────────→ 스레드 수
     CPU 코어 수  최적점  과잉 스레드
  
  적정 스레드 수 (I/O 바운드):
  ≈ CPU 코어 수 * (1 + I/O 대기 시간 / CPU 처리 시간)
  예: 8코어, DB 쿼리 10ms, CPU 1ms
  ≈ 8 * (1 + 10/1) = 88개
```

---

## ⚖️ 트레이드오프

**Blocking I/O의 본질적 한계와 장점**

Blocking I/O는 하나의 연결에 하나의 스레드가 필요하다. 이것이 동시 연결 수를 스레드 수로 제한한다. 하지만 이 단순함이 오히려 장점이 되는 경우가 많다. 코드가 위에서 아래로 읽히고, 예외 처리가 자연스럽고, 스택 트레이스가 직관적이다. 팀이 동기 코드에 익숙하다면 Non-Blocking으로 전환할 때의 복잡도 비용이 처리량 이득보다 클 수 있다.

**Virtual Thread(Java 21)의 등장 의미**

Virtual Thread는 Blocking I/O를 Non-Blocking처럼 만들어준다. JVM이 OS 스레드 블로킹을 감지해 자동으로 OS 스레드에서 분리하고 다른 Virtual Thread를 실행한다. 개발자는 동기 코드를 그대로 쓰면서 Non-Blocking의 처리량을 얻을 수 있다. 이것이 Spring Boot 3.2+에서 Virtual Thread를 권장하는 이유다.

---

## 📌 핵심 정리

```
Blocking read() 흐름:
  데이터 없음 → TASK_INTERRUPTIBLE → 스케줄러가 다른 프로세스 실행
  데이터 도착 → 인터럽트 → Wake-up → 런큐 추가 → CPU 할당 → 복사

C10K 문제:
  스레드 1개 = 연결 1개
  연결 10,000개 = 스레드 10,000개 필요
  → 메모리 폭발 + 컨텍스트 스위칭 오버헤드

Blocking I/O가 적합한 경우:
  ✓ 내부 서비스 (연결 수 < 스레드 풀)
  ✓ JDBC 등 Blocking API 필수
  ✓ 팀이 동기 코드에 익숙

Blocking I/O가 부적합한 경우:
  ✗ 외부 API 다수 호출 (긴 대기)
  ✗ 동시 연결 > 스레드 풀 크기
  ✗ 이벤트 루프 스레드에서 실행 (WebFlux)

대안:
  WebFlux + R2DBC (완전 Non-Blocking)
  Virtual Thread (Java 21, 동기 코드 유지)
  CQRS + 비동기 메시지 (Blocking 격리)

핵심 진단:
  jstack → Thread.State: WAITING 비율
  /proc/<pid>/task/<tid>/status State: S → Blocking 중
```

---

## 🤔 생각해볼 문제

**Q1.** Tomcat 스레드 풀이 200개인데 현재 200개가 모두 `WAITING` 상태다. 새 HTTP 요청이 들어오면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Tomcat의 `Accept Queue`(기본 `acceptCount=100`)에 대기한다. 큐가 차면 새 연결이 거부되거나 클라이언트 측에서 타임아웃이 발생한다. 이 상태에서 처리량을 높이려면 스레드 풀을 늘리거나(단기 해결), DB 쿼리 최적화로 스레드 점유 시간을 줄이거나(근본 해결), Non-Blocking + R2DBC로 전환(구조 변경)해야 한다. `WAITING` 스레드가 많다면 I/O 대기 원인(DB, 외부 API)을 먼저 줄이는 것이 효과적이다.

</details>

**Q2.** WebFlux 이벤트 루프에서 실수로 `Thread.sleep(1000)`을 호출했다. 어떤 일이 벌어지는가?

<details>
<summary>해설 보기</summary>

이벤트 루프 스레드가 1초간 완전히 블로킹된다. 이 스레드가 처리하던 모든 연결의 이벤트가 1초간 처리되지 않는다. Netty는 기본적으로 CPU 코어 수만큼의 이벤트 루프 스레드를 가지므로, 해당 스레드에 배정된 모든 연결이 1초간 응답 없음 상태가 된다. 다른 스레드의 연결은 정상이다. 이것이 WebFlux에서 절대 Blocking 코드를 실행하면 안 되는 이유다. `Schedulers.boundedElastic()`으로 오프로딩해야 한다.

</details>

**Q3.** Java 21 Virtual Thread를 Tomcat에서 활성화했다. JDBC를 사용하는 코드를 변경 없이 그대로 쓸 수 있는가?

<details>
<summary>해설 보기</summary>

**그렇다, 대부분의 경우.** Virtual Thread가 JDBC 블로킹 호출에서 OS 스레드를 자동으로 반납하고 다른 Virtual Thread를 실행한다. 단, `synchronized` 블록 안에서 블로킹이 발생하면 "pinning"으로 OS 스레드를 점유한다. JDBC 드라이버 내부에 `synchronized`가 있으면 이 문제가 발생한다. PostgreSQL 드라이버는 Virtual Thread 친화적으로 개선됐지만, 일부 드라이버는 pinning 문제가 있다. `-Djdk.tracePinnedThreads=full` JVM 옵션으로 pinning 발생 여부를 확인할 수 있다.

</details>

---

**[⬅️ 이전: 파일 디스크립터와 시스템 콜](./01-file-descriptor-syscall.md)** | **[홈으로 🏠](../README.md)** | **[다음: Non-Blocking I/O ➡️](./03-nonblocking-polling.md)**
