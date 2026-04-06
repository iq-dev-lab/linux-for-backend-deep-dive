# 06. I/O 모델이 백엔드에 미치는 영향

## 🎯 핵심 질문

- Tomcat 스레드 풀 모델과 Netty 이벤트 루프 모델은 커널 레벨에서 어떻게 다른가?
- 동시 연결 1000개에서 두 모델의 스레드 수, 메모리, 컨텍스트 스위칭 비용은?
- Java 21 Virtual Thread는 어떤 방식으로 Blocking I/O의 한계를 우회하는가?
- WebFlux가 항상 Tomcat보다 좋은 것은 아니다 — 언제 어떤 것을 선택해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**"WebFlux로 바꿨더니 성능이 떨어졌다"는 사례가 있다.** CPU 바운드 작업(이미지 처리, 암호화)은 I/O 대기가 없어 이벤트 루프의 장점이 없다. 오히려 비동기 오버헤드와 스케줄러 복잡도만 증가한다. I/O 모델의 특성을 워크로드에 맞게 선택해야 한다.

**Spring Boot 3.2에서 Virtual Thread를 활성화하면 JDBC를 그대로 쓸 수 있다는데, 원리가 무엇인가?** Virtual Thread는 JVM 스케줄러가 JDBC 블로킹을 감지해 OS 스레드를 반납하고 다른 Virtual Thread로 전환한다. 개발자는 동기 코드를 쓰지만 OS 스레드 관점에서는 Non-Blocking처럼 동작한다.

**Tomcat의 기본 스레드 풀 200개와 Netty의 이벤트 루프 8개(CPU 코어 수) — 같은 서버에서 어느 쪽이 동시 연결을 더 많이 처리하는가?** I/O 바운드 서비스라면 Netty가 압도적이다. CPU 바운드라면 거의 차이가 없다. 워크로드를 먼저 파악해야 한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```java
// 실수 1: I/O 모델 이해 없이 WebFlux 도입
// 기존 Tomcat 기반 JDBC 코드를 WebFlux로 감쌈
@GetMapping("/users")
public Flux<User> getUsers() {
    return Flux.fromIterable(
        jdbcTemplate.query(...)  // ← JDBC: Blocking!
    );
    // → 이벤트 루프 스레드에서 JDBC 블로킹 발생
    // → 다른 요청 전부 대기
    // → Tomcat보다 처리량 낮아짐
}

// 실수 2: WebFlux에서 CPU 집약 작업
@GetMapping("/compress")
public Mono<byte[]> compress(@RequestBody byte[] data) {
    return Mono.fromCallable(() -> {
        return compress(data);  // CPU 집약 (수십 ms)
        // → 이벤트 루프 스레드 수십 ms 점유
        // → 다른 요청 수십 ms 대기
    });
    // 올바른 처리: .subscribeOn(Schedulers.boundedElastic())
}
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```java
// 워크로드 분류 → 모델 선택

// Case 1: I/O 바운드 + Non-Blocking DB
// → WebFlux + R2DBC
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return userR2dbcRepository.findById(id);  // Non-Blocking
}

// Case 2: I/O 바운드 + JDBC (Blocking 필수)
// → Virtual Thread (Java 21) + Tomcat
// application.properties:
// spring.threads.virtual.enabled=true
// server.tomcat.threads.max=1000  ← 큰 의미 없음 (VT는 OS 스레드 공유)
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return jdbcTemplate.queryForObject(...);  // 자동으로 VT에서 실행
}

// Case 3: CPU 바운드
// → Tomcat (스레드 풀) 또는 WebFlux + boundedElastic
@GetMapping("/heavy")
public Mono<Result> process(@RequestBody Data data) {
    return Mono.fromCallable(() -> heavyCompute(data))
               .subscribeOn(Schedulers.boundedElastic());
}
```

```bash
# Virtual Thread 활성화 확인
$ jstack $PID | grep "VirtualThread" | wc -l
# Virtual Thread 수가 OS 스레드보다 훨씬 많으면 정상

# 이벤트 루프 스레드 상태 (WebFlux)
$ jstack $PID | grep -A 3 "reactor-http-nio"
# RUNNABLE: 요청 처리 중
# WAITING: epoll_wait 대기 중 (정상 유휴 상태)
```

---

## 🔬 내부 동작 원리

### Tomcat(Blocking) vs Netty(Non-Blocking) 커널 레벨 비교

```
=== Tomcat BIO/NIO 스레드 모델 ===

동시 연결 1000개 가정 (스레드 풀 200개):

OS 스레드 상태:
  [스레드  1~200]: TASK_INTERRUPTIBLE (I/O 대기) ← 200개 OS 스레드
  [스레드   201~]: 대기 큐 (연결 대기 중, 수락 못 함)

커널 활동:
  - 200개 스레드의 소켓 대기 큐 관리
  - 패킷 도착마다 Wake-up + 컨텍스트 스위칭
  - 초당 컨텍스트 스위칭: 요청 수 * 2 (수신/송신)

메모리:
  - OS 스레드 스택: 200 * 1MB = 200MB
  - 소켓 버퍼: 1000 * (수신 87KB + 송신 87KB) = 174MB

=== Netty 이벤트 루프 모델 ===

동시 연결 1000개 가정 (이벤트 루프 8개):

OS 스레드 상태:
  [I/O 스레드 1~8]: epoll_wait() 대기 (TASK_INTERRUPTIBLE)
  → 1000개 연결 이벤트를 8개 스레드가 처리

커널 활동:
  - epoll 인스턴스: 1000개 fd 레드-블랙 트리로 관리
  - 패킷 도착: 해당 fd의 epitem을 준비 목록에 추가
  - epoll_wait() Wake-up: 해당 이벤트 루프 스레드만

메모리:
  - OS 스레드 스택: 8 * 1MB = 8MB (25배 절감!)
  - epoll 인스턴스: 1000 * ~수백 바이트 = ~수MB
  - 소켓 버퍼: 1000 * 174KB = 174MB (동일)

컨텍스트 스위칭 비교:
  Tomcat: 초당 수천~수만 회 (200 스레드 경합)
  Netty:  초당 수백 회 (8 스레드, epoll 이벤트 기반)
```

### Java 21 Virtual Thread의 동작 원리

```
Platform Thread (Java 1~20):

  [Virtual(=Platform) Thread] = [OS Thread] (1:1 매핑)
  JDBC 블로킹 → OS 스레드 블로킹 → 스케줄러 대기 큐

Virtual Thread (Java 21):

  [Virtual Thread 1] ──┐
  [Virtual Thread 2] ──┤  JVM ForkJoinPool
  [Virtual Thread 3] ──┤  [OS Thread 1] [OS Thread 2] ... [OS Thread N]
  ...                  │  (N = CPU 코어 수)
  [Virtual Thread 1M] ─┘

JDBC 블로킹 발생 시:
  1. VT가 JDBC read() 호출 (Blocking 소켓)
  2. JVM이 OS 스레드 블로킹 감지
  3. VT를 OS 스레드에서 분리 (unmount)
  4. 다른 VT를 OS 스레드에 mount → 계속 실행
  5. JDBC 응답 도착 → VT를 다시 OS 스레드에 mount
  6. VT가 결과 처리 계속

결과:
  JDBC 대기 시간 동안 OS 스레드는 다른 VT 처리
  → 1만 개의 동시 JDBC 요청도 8개 OS 스레드로 처리 가능
  → 개발자는 동기 코드 그대로 작성

VT의 한계 (Pinning):
  synchronized 블록 안에서 블로킹 발생 시
  → OS 스레드를 반납하지 못함 (Pinned 상태)
  → 진단: -Djdk.tracePinnedThreads=full
```

### 워크로드별 최적 모델 선택

```
I/O 바운드 + 짧은 응답 시간:
  대기: DB 쿼리 10ms, CPU 처리 1ms
  최적: WebFlux + R2DBC (이벤트 루프)
  이유: 대기 중 OS 스레드 반납 → 높은 동시성

I/O 바운드 + JDBC 필수:
  대기: DB 쿼리 10ms, CPU 처리 1ms
  최적: Spring MVC + Virtual Thread
  이유: JDBC 블로킹이 OS 스레드 차단 안 함

I/O 바운드 + 외부 API 호출 많음:
  대기: 외부 API 200ms, CPU 처리 5ms
  최적: WebFlux + WebClient (Non-Blocking HTTP)
  이유: 200ms 대기 시 이벤트 루프가 다른 요청 처리

CPU 바운드:
  예: 이미지 처리, 암호화, 복잡한 계산
  최적: Spring MVC + 스레드 풀 (CPU 코어 수 * 2)
  이유: Non-Blocking이 이득 없음, 복잡도만 증가

혼합 (CPU + I/O):
  최적: WebFlux + boundedElastic 스케줄러로 CPU 작업 오프로딩
```

### 실제 동시 연결 한계 계산

```
서버 스펙: 16코어, 32GB RAM

=== Tomcat 기반 계산 ===
스레드 1개 비용:
  OS 스레드 스택: 1MB (기본)
  JVM 메타데이터: ~100KB
  합계: ~1.1MB

최대 스레드 수:
  32GB / 1.1MB ≈ 29,000개 (메모리 기준)
  실용적 한도: 1,000~2,000개 (컨텍스트 스위칭 고려)

동시 처리 가능 요청 (JDBC 10ms 기준):
  스레드 1,000개 * (1초 / 10ms) = 100,000 req/s (이론값)
  실제: 컨텍스트 스위칭 오버헤드로 50,000 req/s 수준

=== Netty 이벤트 루프 기반 계산 ===
이벤트 루프 스레드: 16~32개 (CPU 코어 수 * 1~2)
OS 스레드 메모리: 32개 * 1MB = 32MB (미미함)
동시 연결 제한: 메모리 아닌 소켓 버퍼 기준
  소켓 1개 버퍼: 128KB~256KB
  32GB / 256KB ≈ 131,000개 (이론값)

동시 처리 가능 요청 (R2DBC 10ms):
  이벤트 루프 32개 * (1초 / 0.001ms 이벤트 처리) ≈ 수백만 req/s
  DB 연결 풀 크기에 실질적으로 제한됨
```

---

## 💻 실전 실험

### 실험 1: Tomcat vs WebFlux 처리량 비교

```bash
# Tomcat 기반 Spring MVC (JDBC, I/O 바운드 시뮬레이션)
$ cat > TomcatController.java << 'EOF'
@GetMapping("/api/slow")
public String slow() throws Exception {
    Thread.sleep(10);  // DB 10ms 시뮬레이션
    return "ok";
}
EOF

# WebFlux 기반 (Non-Blocking 시뮬레이션)
$ cat > WebFluxController.java << 'EOF'
@GetMapping("/api/slow")
public Mono<String> slow() {
    return Mono.just("ok").delayElement(Duration.ofMillis(10));
}
EOF

# 동시 요청 500개로 부하 테스트
$ wrk -t4 -c500 -d30s http://localhost:8080/api/slow
# Tomcat (스레드 200개):
#   Requests/sec: ~18,000 (200 스레드 * 100 req/s each)
# WebFlux:
#   Requests/sec: ~48,000 (Non-Blocking, 이벤트 루프 효율)

# OS 스레드 수 비교
$ cat /proc/$(pgrep java)/status | grep Threads
# Tomcat: ~230개 (스레드 풀 + JVM 내부)
# WebFlux: ~50개 (이벤트 루프 + JVM 내부)
```

### 실험 2: Virtual Thread 동작 확인 (Java 21)

```java
// VirtualThreadDemo.java
import java.util.concurrent.*;
import java.time.*;

public class VirtualThreadDemo {
    static void simulateJdbc() throws Exception {
        Thread.sleep(10);  // JDBC 블로킹 시뮬레이션
    }

    public static void main(String[] args) throws Exception {
        int requests = 10_000;

        // Platform Thread 풀 (200개)
        long start = System.currentTimeMillis();
        ExecutorService platformPool = Executors.newFixedThreadPool(200);
        CountDownLatch latch1 = new CountDownLatch(requests);
        for (int i = 0; i < requests; i++) {
            platformPool.submit(() -> {
                try { simulateJdbc(); } catch (Exception e) {}
                latch1.countDown();
            });
        }
        latch1.await();
        System.out.printf("Platform Thread: %dms%n",
            System.currentTimeMillis() - start);

        // Virtual Thread
        start = System.currentTimeMillis();
        ExecutorService virtualPool = Executors.newVirtualThreadPerTaskExecutor();
        CountDownLatch latch2 = new CountDownLatch(requests);
        for (int i = 0; i < requests; i++) {
            virtualPool.submit(() -> {
                try { simulateJdbc(); } catch (Exception e) {}
                latch2.countDown();
            });
        }
        latch2.await();
        System.out.printf("Virtual Thread:  %dms%n",
            System.currentTimeMillis() - start);
    }
}
```

```bash
$ java --version  # 21+ 확인
$ java VirtualThreadDemo
Platform Thread: 510ms  (200스레드로 10,000요청 처리, 50배치)
Virtual Thread:   28ms  (거의 동시 처리)
```

### 실험 3: 이벤트 루프 스레드 블로킹 감지

```bash
# WebFlux 이벤트 루프 스레드 블로킹 감지
$ jstack $PID | grep -A 15 "reactor-http-nio"
# 정상:
# "reactor-http-nio-1" ... RUNNABLE
#   sun.nio.ch.EPollArrayWrapper.epollWait  ← epoll_wait 대기 중

# 문제 상황 (Thread.sleep 호출 시):
# "reactor-http-nio-1" ... TIMED_WAITING
#   java.lang.Thread.sleep  ← 이벤트 루프 블로킹!

# Reactor 블로킹 감지 도구
# build.gradle: testImplementation 'io.projectreactor.tools:blockhound:1.0.8.RELEASE'
# @Test 또는 개발 환경에서:
BlockHound.install();
# 이벤트 루프에서 Blocking 호출 시 BlockingOperationError 발생
```

---

## 📊 성능/비용 비교

**동시 연결 1000개 기준 리소스 사용 비교**:

```
                    Tomcat        Netty/WebFlux   Virtual Thread
OS 스레드 수        200~1000개    8~16개          8~16개 (VT 별도)
스레드 스택 메모리  200~1000MB   8~16MB          8~16MB
컨텍스트 스위칭    초당 수만 회  초당 수백 회    초당 수백 회
코드 스타일         동기          비동기(Reactor) 동기
JDBC 호환성         완벽          불가 (R2DBC)    완벽
CPU 바운드 성능     좋음          동일            동일
I/O 바운드 처리량   중간          높음            높음
학습 곡선           낮음          높음 (Reactor)  낮음
```

**I/O 바운드 처리량 공식**:

```
Tomcat:
  처리량 = 스레드 수 / I/O 대기 시간
  예: 200 스레드 / 0.01초(10ms) = 20,000 req/s

WebFlux:
  처리량 = 이론적으로 I/O 대기 시간과 무관
  실질 제한: DB 연결 풀, 외부 서비스 용량
  예: 동일 DB 100 연결 / 0.01초 = 10,000 req/s (DB가 병목)
  → DB 확장 시 선형 증가, Tomcat은 스레드 확장 필요
```

---

## ⚖️ 트레이드오프

**WebFlux의 진짜 복잡성**

WebFlux(Reactor)의 비동기 프로그래밍 모델은 학습 곡선이 가파르다. 디버깅도 어렵다. 스택 트레이스가 콜백 체인으로 분산되어 오류 원인을 찾기 어렵다. 팀 전체가 Reactor를 숙지하지 않으면 유지보수 비용이 급증한다. 성능 이득이 이 비용을 정당화하는지 냉정하게 판단해야 한다.

**Virtual Thread가 WebFlux를 대체하는가**

Virtual Thread는 I/O 바운드 + Blocking API(JDBC 등) 시나리오에서 WebFlux와 비슷한 처리량을 훨씬 단순한 코드로 달성한다. 하지만 완전한 Non-Blocking 스택(R2DBC, WebClient)과 조합한 WebFlux가 여전히 리소스 효율이 높다. 팀의 역량과 기존 스택에 따라 선택이 달라진다.

**적정선의 결정**

내부 마이크로서비스 간 통신이고 동시 연결이 수백 개라면 Tomcat으로도 충분하다. 외부 사용자 트래픽을 받는 API 게이트웨이나 실시간 스트리밍 서비스라면 Netty/WebFlux가 적합하다. 과도하게 최적화하지 말고, 실제 병목이 확인된 후에 전환을 결정하라.

---

## 📌 핵심 정리

```
I/O 모델 선택 기준:
  CPU 바운드   → Tomcat (스레드 풀) or Virtual Thread
  I/O 바운드   → WebFlux + R2DBC or Virtual Thread + JDBC
  혼합          → WebFlux + boundedElastic 오프로딩

Tomcat vs Netty 리소스 차이 (1000연결):
  Tomcat: 200~1000 OS 스레드, 200MB~1GB 스택
  Netty:  8~16 OS 스레드, 8~16MB 스택
  차이:   컨텍스트 스위칭 10~100배, 메모리 10~50배

Virtual Thread (Java 21):
  동기 코드로 Non-Blocking 처리량
  JDBC 블로킹 자동 우회
  synchronized 안 블로킹 = Pinning 주의

WebFlux 피해야 할 경우:
  ✗ JDBC + WebFlux 조합 (블로킹 오프로딩 필수)
  ✗ CPU 바운드 작업 (이벤트 루프에서 직접 실행)
  ✗ 팀이 Reactor 모르는 경우

진단 명령어:
  jstack → Thread.State 분포 (WAITING 비율)
  /proc/<pid>/status Threads → OS 스레드 수
  top sy → 컨텍스트 스위칭 오버헤드
  vmstat cs → 초당 컨텍스트 스위칭 수
```

---

## 🤔 생각해볼 문제

**Q1.** 외부 결제 API를 호출하는 서비스가 있다. 결제 API 응답이 평균 500ms다. Tomcat 스레드 풀 200개와 WebFlux 이벤트 루프 8개 중 어느 쪽이 더 많은 동시 결제 요청을 처리할 수 있는가?

<details>
<summary>해설 보기</summary>

**WebFlux가 압도적으로 유리하다.** Tomcat 기준으로 초당 최대 처리량은 `200 / 0.5초 = 400 req/s`다. 스레드 200개가 모두 500ms 동안 외부 API 응답을 기다리기 때문이다. WebFlux는 Non-Blocking WebClient를 사용하면 이벤트 루프 8개가 수천 개의 동시 API 호출을 관리할 수 있다. 실제 처리량은 외부 API의 용량에 제한되지만, 클라이언트 쪽에서는 서버 리소스 소비 없이 수만 개의 요청을 동시에 대기시킬 수 있다.

</details>

**Q2.** Virtual Thread를 활성화한 Spring Boot에서 `@Transactional`이 붙은 메서드 내에서 Virtual Thread가 여러 OS 스레드로 이동할 수 있는가?

<details>
<summary>해설 보기</summary>

**원칙적으로 이동 가능하지만, 트랜잭션 안전성은 유지된다.** Virtual Thread는 블로킹 지점에서 OS 스레드를 바꿀 수 있다. Spring의 `@Transactional`은 트랜잭션 컨텍스트를 `ThreadLocal`에 저장하는데, Virtual Thread는 자신만의 `ThreadLocal`을 가지므로 OS 스레드가 바뀌어도 트랜잭션 컨텍스트는 유지된다. 단, `synchronized` 블록이나 native 코드에서 Pinning이 발생하면 OS 스레드를 고정한 채로 트랜잭션이 진행된다.

</details>

**Q3.** Netty 이벤트 루프가 8개 스레드로 10,000 연결을 처리한다. 갑자기 이벤트 루프 스레드 중 하나에서 긴 CPU 작업이 실행됐다. 이 스레드가 담당하는 연결들은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

해당 이벤트 루프 스레드가 담당하는 약 1,250개 연결(10,000 / 8)이 CPU 작업이 완료될 때까지 응답을 받지 못한다. 이벤트 루프는 단일 스레드이므로 한 태스크가 실행되는 동안 다른 이벤트는 큐에 대기한다. CPU 바운드 작업을 반드시 `Schedulers.boundedElastic()`이나 별도 스레드 풀로 오프로딩해야 하는 이유다. 이것이 WebFlux 서버에서 이벤트 루프 스레드 블로킹이 치명적인 이유이기도 하다.

</details>

---

**[⬅️ 이전: epoll](./05-epoll.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — VFS ➡️](../filesystem-disk-io/01-vfs.md)**
