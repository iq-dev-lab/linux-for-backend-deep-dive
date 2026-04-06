# 03. 스레드 모델 — OS 스레드와 1:1 매핑

## 🎯 핵심 질문

- 스레드는 프로세스와 커널 레벨에서 무엇이 다른가?
- 스레드 생성이 프로세스 생성보다 빠른 구체적인 이유는?
- Java의 스레드는 OS 스레드와 어떻게 매핑되는가?
- Virtual Thread(Java 21)는 OS 스레드 모델의 어떤 한계를 해결하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Tomcat의 기본 스레드 풀이 200개인 이유는?** 스레드 하나가 OS 스레드와 1:1 매핑되고, OS 스레드는 생성 비용(메모리, 시간)이 크기 때문에 미리 만들어두는 것이다. 이 한계를 알아야 스레드 풀 크기를 올바르게 설정할 수 있다.

**Spring WebFlux로 전환했더니 처리량이 늘었다.** 이벤트 루프 기반 모델은 소수의 OS 스레드로 수만 개의 동시 연결을 처리한다. 왜 그것이 가능한지는 OS 스레드의 비용 구조를 알아야 이해된다.

**Java 21 Virtual Thread가 스레드 풀을 없앨 수 있다는 말이 무슨 의미인가?** Virtual Thread는 OS 스레드와 1:1 매핑을 끊고, 소수의 OS 스레드 위에서 수백만 개의 가상 스레드를 실행한다. OS 스레드 모델의 한계를 알지 못하면 이것이 왜 혁신적인지 이해할 수 없다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```java
// 상황: 처리량을 높이기 위해 스레드 풀 크기를 무작정 늘린다
@Configuration
public class ThreadPoolConfig {
    @Bean
    public ThreadPoolExecutor executor() {
        // "스레드가 많을수록 처리량이 높아지겠지"
        return new ThreadPoolExecutor(1000, 2000, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>());
    }
}

// 결과:
// 1. 스레드 2000개 * 스택 1MB = 2GB 메모리 소비
// 2. 컨텍스트 스위칭 폭발 → CPU `sy` 사용률 급증
// 3. 오히려 처리량 감소
```

```bash
# 확인해보면:
$ top
%Cpu(s): 5.0 us, 40.0 sy, 0.0 ni, 54.0 id ...
#                ↑↑↑↑↑↑↑↑↑↑
# sy(system) 40%: 컨텍스트 스위칭에 CPU를 소비 중
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 현재 Java 프로세스의 스레드 수와 상태 확인
$ cat /proc/$PID/status | grep Threads
Threads: 48

# 스레드 목록과 각 스레드 상태
$ ls /proc/$PID/task/   # 각 디렉토리가 스레드 하나
12345  12346  12347 ...

# 각 스레드의 상태 확인
$ for tid in $(ls /proc/$PID/task/); do
    echo -n "TID $tid: "
    cat /proc/$PID/task/$tid/status | grep "^State"
  done

# 적정 스레드 수 판단:
# CPU 바운드 작업: CPU 코어 수 + 1
# I/O 바운드 작업: CPU 코어 수 * (1 + I/O 대기 시간 / CPU 사용 시간)
```

```java
// Virtual Thread 활용 (Java 21+)
// OS 스레드 풀 크기에 제한받지 않음
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            // I/O 블로킹 작업도 OS 스레드를 점유하지 않음
            Thread.sleep(Duration.ofMillis(100));
        });
    }
}
// OS 스레드는 CPU 코어 수만큼만 사용
```

---

## 🔬 내부 동작 원리

### 리눅스에서 스레드는 프로세스다

리눅스 커널에서 스레드와 프로세스는 모두 `task_struct`로 표현된다. 차이는 **자원을 공유하는 정도**뿐이다.

```
프로세스 생성 (fork):
  새 task_struct 생성
  + 새 mm_struct (독립된 가상 주소 공간)
  + 새 파일 디스크립터 테이블
  + 새 시그널 핸들러 테이블

스레드 생성 (clone with CLONE_VM | CLONE_FILES | CLONE_SIGHAND):
  새 task_struct 생성
  + 부모의 mm_struct 공유 (가상 주소 공간 공유!)
  + 부모의 파일 디스크립터 테이블 공유
  + 부모의 시그널 핸들러 테이블 공유
  + 독립된 스택 (새로 할당)
```

`clone()` 시스템 콜에 어떤 플래그를 주느냐에 따라 프로세스/스레드/컨테이너가 결정된다:

```c
/* 스레드 생성 */
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD, ...)

/* 프로세스 생성 (fork와 동일) */
clone(SIGCHLD, ...)

/* 컨테이너 생성 (namespace 분리) */
clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | SIGCHLD, ...)
```

### 스레드가 공유하는 것 vs 독립적인 것

```
같은 프로세스 내 스레드들:

  ┌─────────────────────────────────────┐
  │           프로세스 주소 공간            │
  │  ┌──────┐ ┌──────┐ ┌──────┐         │
  │  │코드   │ │ 힙    │ │ 데이터│         │  ← 모든 스레드가 공유
  │  └──────┘ └──────┘ └──────┘         │
  │                                     │
  │  ┌──────┐ ┌──────┐ ┌──────┐         │
  │  │스택1  │ │스택2  │ │스택3   │        │  ← 각 스레드 독립
  │  │(T1)  │ │(T2)  │ │(T3)  │         │
  │  └──────┘ └──────┘ └──────┘         │
  └─────────────────────────────────────┘

공유 자원:
  ✓ 코드(.text), 데이터(.data/.bss), 힙
  ✓ 파일 디스크립터 (소켓, 파일)
  ✓ 시그널 핸들러
  ✓ 현재 작업 디렉토리

독립 자원:
  ✗ 스택 (지역 변수, 함수 호출 체인)
  ✗ CPU 레지스터 상태 (rip, rsp 등)
  ✗ 스레드 로컬 스토리지 (TLS)
  ✗ 시그널 마스크
```

### 스레드 생성 비용 비교

```
프로세스 생성 (fork):
  1. task_struct 할당            ~ 수 마이크로초
  2. mm_struct 복사 (페이지 테이블)  ~ 수십~수백 마이크로초 (메모리 크기 비례)
  3. 파일 디스크립터 테이블 복사    ~ 마이크로초
  4. CoW 설정 (모든 페이지 읽기 전용)~ 수십~수백 마이크로초

스레드 생성 (clone with CLONE_VM):
  1. task_struct 할당            ~ 수 마이크로초
  2. mm_struct 참조 카운트 증가    ~ 나노초 (복사 없음!)
  3. 새 스택 할당 (보통 8MB 예약)  ~ 마이크로초
  4. TLB 플러시 불필요            ← 핵심! 같은 페이지 테이블 공유

결론: 스레드 생성 ≈ 10~100배 빠름
```

### Java 스레드와 OS 스레드의 관계

```
Java 1.1 ~ Java 20 (Platform Thread):

  [Java Thread 1] ──1:1 매핑──→ [OS Thread 1]
  [Java Thread 2] ──1:1 매핑──→ [OS Thread 2]
  [Java Thread N] ──1:1 매핑──→ [OS Thread N]

  한계:
  - OS 스레드 1개 ≈ 메모리 1~8MB (스택)
  - 수천 개 이상은 컨텍스트 스위칭 오버헤드
  - I/O 블로킹 시 OS 스레드 점유 → 낭비

Java 21 Virtual Thread:

  [VThread 1]  [VThread 2]  [VThread 3] ... [VThread 100만]
       ↓              ↓              ↓
  ┌──────────────────────────────────┐
  │  ForkJoinPool (OS Thread Pool)   │
  │  [OS Thread 1] [OS Thread 2] ... │  ← CPU 코어 수만큼
  └──────────────────────────────────┘

  VThread가 I/O 대기 시:
    - OS 스레드에서 분리 (unmount)
    - 다른 VThread가 OS 스레드 사용
    - I/O 완료 시 다시 OS 스레드에 할당 (mount)
```

---

## 💻 실전 실험

### 실험 1: 스레드 목록과 상태 확인

```bash
$ PID=$(pgrep -f "spring-boot") # Spring Boot PID 확인

# 스레드 목록 (task 디렉토리)
$ ls /proc/$PID/task/ | wc -l
48   ← 현재 스레드 수

# 각 스레드의 이름과 상태
$ for tid in $(ls /proc/$PID/task/ | head -10); do
    name=$(cat /proc/$PID/task/$tid/comm 2>/dev/null)
    state=$(awk '/State:/{print $2,$3}' /proc/$PID/task/$tid/status 2>/dev/null)
    echo "TID=$tid Name=$name State=$state"
  done

# 출력 예:
# TID=12345 Name=main       State=S (sleeping)
# TID=12346 Name=http-nio-1 State=S (sleeping)
# TID=12347 Name=http-nio-2 State=R (running)
```

```bash
# Java jstack으로 스레드 덤프 (더 상세)
$ jstack $PID | grep -E "^\"" | awk -F'"' '{print $2}' | head -20
```

### 실험 2: 스레드 생성 비용 측정

```c
/* thread_vs_process.c */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <time.h>
#include <sys/wait.h>

void *thread_func(void *arg) { return NULL; }

int main() {
    struct timespec start, end;
    int N = 1000;

    /* 스레드 생성 N번 */
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < N; i++) {
        pthread_t t;
        pthread_create(&t, NULL, thread_func, NULL);
        pthread_join(t, NULL);
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    long thread_ns = (end.tv_sec - start.tv_sec) * 1e9 + (end.tv_nsec - start.tv_nsec);

    /* 프로세스 생성 N번 */
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < N; i++) {
        pid_t pid = fork();
        if (pid == 0) exit(0);
        waitpid(pid, NULL, 0);
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    long process_ns = (end.tv_sec - start.tv_sec) * 1e9 + (end.tv_nsec - start.tv_nsec);

    printf("스레드 생성 %d회: %ld ms (평균 %ld μs)\n",
           N, thread_ns/1000000, thread_ns/1000/N);
    printf("프로세스 생성 %d회: %ld ms (평균 %ld μs)\n",
           N, process_ns/1000000, process_ns/1000/N);
    printf("프로세스/스레드 비율: %.1fx\n", (double)process_ns/thread_ns);
    return 0;
}
```

```bash
$ gcc thread_vs_process.c -o tvp -lpthread && ./tvp
# 예상 출력:
# 스레드 생성 1000회: 150 ms (평균 150 μs)
# 프로세스 생성 1000회: 800 ms (평균 800 μs)
# 프로세스/스레드 비율: 5.3x
```

### 실험 3: Virtual Thread 동작 확인 (Java 21+)

```java
// VirtualThreadDemo.java
import java.util.concurrent.Executors;
import java.time.Duration;

public class VirtualThreadDemo {
    public static void main(String[] args) throws Exception {
        System.out.println("플랫폼 스레드 테스트:");
        long start = System.currentTimeMillis();

        // Platform Thread: 1000개 동시 I/O → OS 스레드 1000개 필요
        try (var exec = Executors.newFixedThreadPool(1000)) {
            for (int i = 0; i < 10000; i++) {
                exec.submit(() -> {
                    Thread.sleep(Duration.ofMillis(100));
                    return null;
                });
            }
        }
        System.out.println("Platform: " + (System.currentTimeMillis() - start) + "ms");

        start = System.currentTimeMillis();
        // Virtual Thread: 10000개 동시 I/O → OS 스레드는 코어 수만
        try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 10000; i++) {
                exec.submit(() -> {
                    Thread.sleep(Duration.ofMillis(100));
                    return null;
                });
            }
        }
        System.out.println("Virtual: " + (System.currentTimeMillis() - start) + "ms");
    }
}
```

```bash
$ java --version  # Java 21+ 확인
$ java VirtualThreadDemo
# Platform: ~1100ms (1000개 풀로 10000개 처리, 10번 배치)
# Virtual:  ~120ms  (10000개 거의 동시 처리)
```

---

## 📊 성능/비용 비교

| 항목 | 플랫폼 스레드 (Java 1~20) | Virtual Thread (Java 21+) |
|------|--------------------------|--------------------------|
| OS 스레드 매핑 | 1:1 | M:N (다수:소수) |
| 메모리 (스택) | ~1MB/스레드 | ~수 KB/스레드 |
| 생성 비용 | 수백 μs | 수 μs |
| I/O 대기 시 OS 스레드 | 점유 (낭비) | 반납 (효율적) |
| 동시 실행 한계 | 수천 (메모리/컨텍스트 스위칭) | 수백만 (이론상) |
| CPU 바운드 성능 | 동일 | OS 스레드 수에 제한 |

**스레드 수와 처리량의 관계**:

```
처리량
  ↑
  │         ▲ 최적점
  │        /│\
  │       / │ \   ← 컨텍스트 스위칭 오버헤드 증가
  │      /  │  \
  │     /   │   \
  │    /    │    ─────────
  │   /     │
  └───────────────────────→ 스레드 수
          CPU 코어 수 * 2~3 (I/O 바운드 기준 추정)
```

---

## ⚖️ 트레이드오프

**많은 스레드의 장점과 단점**:

장점: I/O 대기 중인 스레드가 많을수록 CPU를 더 활용할 수 있다. 스레드 하나가 블로킹 I/O를 기다리는 동안 다른 스레드가 CPU를 사용한다.

단점: 스레드 수가 과도하면 컨텍스트 스위칭 비용이 폭발한다. CPU가 레지스터 저장/복원, TLB 플러시, 캐시 무효화에 시간을 쏟는다. `top`에서 `sy`(시스템) CPU 사용률이 높아진다.

**Virtual Thread의 트레이드오프**:

Virtual Thread는 I/O 바운드 작업에서 탁월하다. 하지만 CPU 바운드 작업(암호화, 이미지 처리 등)에서는 OS 스레드 수(= CPU 코어 수)에 여전히 제한된다. 또한 `synchronized` 블록 안에서 I/O 대기 시 "pinning" 문제가 발생해 OS 스레드를 점유할 수 있다.

---

## 📌 핵심 정리

```
리눅스에서 스레드 = 주소 공간 공유하는 가벼운 프로세스
  - 내부적으로 clone(CLONE_VM | ...) 시스템 콜
  - 프로세스보다 생성 빠름: 페이지 테이블 복사 없음 (공유)

스레드가 공유하는 것: 코드, 힙, 데이터, 파일 디스크립터
스레드가 독립적인 것: 스택, CPU 레지스터, TLS

Java 스레드 모델:
  Java 1~20: Platform Thread = OS Thread (1:1)
    → 스레드 많을수록 메모리 + 컨텍스트 스위칭 비용
    → Tomcat 기본 200개 제한의 이유
  Java 21+: Virtual Thread = OS Thread에 M:N 매핑
    → I/O 대기 시 OS 스레드 반납 → 수백만 동시 처리 가능

핵심 진단:
  /proc/<pid>/task/          → 스레드 수 확인
  /proc/<pid>/task/<tid>/status → 스레드별 상태
  top에서 sy 높음            → 컨텍스트 스위칭 과다
  jstack <pid>              → Java 스레드 덤프
```

---

## 🤔 생각해볼 문제

**Q1.** Tomcat의 스레드 풀 크기가 200인데, 현재 커넥션이 500개 들어온다. 나머지 300개 요청은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Tomcat의 `acceptCount`(기본 100) 큐에 대기한다. 큐도 차면 새 연결은 `Connection refused` 또는 타임아웃을 받는다. 이때 스레드 수를 무작정 늘리면 컨텍스트 스위칭 비용으로 오히려 처리량이 감소한다. 올바른 접근은 I/O 바운드 작업이라면 Virtual Thread 도입, CPU 바운드라면 비동기 처리나 수평 확장을 검토하는 것이다.

</details>

**Q2.** 멀티스레드 프로그램에서 한 스레드가 `malloc()`으로 힙에 메모리를 할당했다. 다른 스레드도 이 메모리를 접근할 수 있는가?

<details>
<summary>해설 보기</summary>

**그렇다.** 같은 프로세스 내 스레드는 힙 메모리를 공유한다. 포인터를 다른 스레드에 전달하면 즉시 접근 가능하다. 이것이 스레드 간 데이터 공유가 쉬운 이유이자, 동시에 레이스 컨디션과 데이터 경합이 발생하는 이유다. 프로세스 간에는 이 공유가 불가능하므로 IPC(파이프, 공유 메모리 등)가 필요하다.

</details>

**Q3.** Virtual Thread를 사용하면 `synchronized` 키워드 사용에 주의가 필요하다. 왜인가?

<details>
<summary>해설 보기</summary>

Virtual Thread가 `synchronized` 블록 안에서 I/O 대기 상태가 되면 "pinning"이 발생한다. 이 경우 Virtual Thread가 OS 스레드를 계속 점유하게 되어, Virtual Thread의 핵심 장점(I/O 대기 시 OS 스레드 반납)이 사라진다. 해결책은 `synchronized` 대신 `ReentrantLock`을 사용하거나, `synchronized` 블록을 최대한 짧게 유지하는 것이다. JDK 팀이 향후 버전에서 이 제한을 완화할 예정이다.

</details>

---

**[⬅️ 이전: fork()와 exec()](./02-fork-exec-cow.md)** | **[홈으로 🏠](../README.md)** | **[다음: 컨텍스트 스위칭 비용 ➡️](./04-context-switching.md)**
