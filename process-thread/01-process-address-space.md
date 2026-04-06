# 01. 프로세스란 무엇인가 — 주소 공간과 PCB

## 🎯 핵심 질문

- 프로세스는 프로그램과 어떻게 다른가?
- 코드/데이터/힙/스택 세그먼트는 가상 주소 공간에서 어디에 위치하는가?
- PCB(Process Control Block)는 무엇을 저장하고, 컨텍스트 스위칭과 어떻게 연결되는가?
- `/proc/<pid>/maps`를 읽으면 어떤 정보를 얻을 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"프로세스가 뭔지 모르는 개발자"는 거의 없다. 하지만 "프로세스의 주소 공간이 어떻게 생겼는지 설명할 수 있는 개발자"는 드물다.

이 차이가 실무에서 드러나는 순간들이 있다.

**Java 애플리케이션이 `-Xmx4g`인데 실제 메모리는 6GB를 쓴다.** Heap 외에도 스택, 메타스페이스, 코드 캐시, Native Memory가 존재한다는 사실을 모르면 이 상황을 진단할 수 없다. `/proc/<pid>/maps`를 보면 JVM이 주소 공간을 어떻게 사용하는지 한눈에 보인다.

**Redis가 메모리 사용량을 `MEMORY USAGE` 명령으로 확인했는데 `free -h`의 used와 다르다.** Redis는 jemalloc으로 메모리를 관리하고, 프로세스 주소 공간 전체가 아닌 데이터 구조에 할당된 부분만 보고한다. 커널이 프로세스에게 실제로 어떤 메모리를 매핑했는지는 `/proc/<pid>/smaps`에서 확인해야 한다.

**`kill -9`를 썼는데 프로세스가 안 죽는다.** `SIGKILL`은 커널이 직접 처리하므로 반드시 죽는다. 안 죽는 건 `D` 상태(Uninterruptible Sleep)로 I/O 대기 중인 것이다. 이를 이해하려면 프로세스 상태와 PCB를 알아야 한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 상황: Java 서버가 OOM으로 죽었다. 메모리를 확인하려 한다.
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           15Gi        12Gi       512Mi       256Mi       2.5Gi       2.8Gi

# "-Xmx는 8g인데 왜 12GB가 사용 중이지?"
# → JVM 프로세스의 어느 영역이 얼마를 쓰는지 모른다
# → Heap만 메모리라고 생각한다
# → 정작 Native Memory 누수를 놓친다
```

또는:

```java
// Spring Boot 서비스. 스레드가 많아 메모리가 걱정된다.
// "스레드 수를 줄이면 메모리가 줄겠지?"
// → 스레드 하나당 스택 메모리(기본 512KB~1MB)를 쓴다는 걸 모른다
// → 200개 스레드 = 최대 200MB 스택만으로도 의미있는 메모리 소비
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# Java 프로세스 메모리 전체 구조를 직접 본다
$ cat /proc/$PID/maps | awk '{print $6}' | sort | uniq -c | sort -rn | head -20

# smaps로 각 영역의 실제 물리 메모리 사용량 확인
$ grep -A 15 "\[heap\]" /proc/$PID/smaps

# Java Native Memory Tracking으로 영역별 분류
$ java -XX:NativeMemoryTracking=summary -jar app.jar
$ jcmd $PID VM.native_memory summary
```

프로세스 주소 공간을 이해하면 메모리 문제를 계층적으로 접근할 수 있다: Heap → Metaspace → 스레드 스택 → Code Cache → Native Memory 순으로 확인한다.

---

## 🔬 내부 동작 원리

### 프로그램 vs 프로세스

**프로그램**은 디스크에 저장된 실행 파일(ELF, JAR 등)이다. 정적인 데이터 덩어리에 불과하다.

**프로세스**는 프로그램이 메모리에 올라와 실행 중인 상태다. 커널은 프로세스마다 독립된 **가상 주소 공간(Virtual Address Space)**을 부여한다.

### 가상 주소 공간 레이아웃

64비트 리눅스에서 프로세스의 가상 주소 공간은 다음과 같이 구성된다:

```
높은 주소 (0xFFFFFFFFFFFFFFFF)
┌─────────────────────────────┐
│         커널 공간             │  ← 모든 프로세스가 공유 (유저 접근 불가)
│    (128TB, 유저 접근 불가)     │
├─────────────────────────────┤ ← 0x00007FFFFFFFFFFF
│      스택 (Stack)            │  ← 위에서 아래로 성장 ↓
│   (함수 호출, 지역변수)         │    스레드마다 독립 존재
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│           ↓ 성장             │
│                             │
│   Memory-Mapped Region      │  ← mmap(), 공유 라이브러리 (.so)
│  (공유 라이브러리, 파일 매핑)     │
│                             │
│           ↑ 성장             │
├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│        힙 (Heap)            │  ← 아래서 위로 성장 ↑
│    (malloc/new 동적 할당)     │
├─────────────────────────────┤
│  초기화된 데이터 (.data)        │  ← 전역 변수 (초기값 있음)
├─────────────────────────────┤
│ 초기화 안 된 데이터 (.bss)       │  ← 전역 변수 (초기값 없음, 0으로 초기화)
├─────────────────────────────┤
│        코드 (.text)          │  ← 실행 가능한 명령어 (읽기 전용)
└─────────────────────────────┘
낮은 주소 (0x0000000000000000)
```

각 세그먼트의 역할:

**코드 세그먼트 (.text)**: 컴파일된 기계어 명령어가 저장된다. 읽기 전용(Read-Only)으로 보호된다. 여러 프로세스가 같은 프로그램을 실행할 때 코드 세그먼트는 물리 메모리를 공유한다(Copy-On-Write 덕분).

**데이터 세그먼트 (.data / .bss)**: 전역 변수와 정적 변수가 저장된다. `.data`는 초기값이 있는 변수, `.bss`는 0으로 초기화되는 변수다.

**힙(Heap)**: `malloc()`/`new`로 동적 할당되는 메모리 영역. `brk()` 또는 `mmap()` 시스템 콜로 크기가 조정된다. Java의 Heap은 실제로 JVM이 `mmap()`으로 예약한 메모리 영역이다.

**스택(Stack)**: 함수 호출 시 스택 프레임이 쌓이는 영역. 지역 변수, 함수 파라미터, 리턴 주소가 저장된다. **스레드마다 독립된 스택을 가진다.** 기본 크기는 보통 8MB(`ulimit -s`로 확인 가능).

**Memory-Mapped Region**: `mmap()` 시스템 콜로 매핑된 파일이나 공유 라이브러리. Java의 `.jar` 파일 내 클래스도 여기에 매핑된다.

### PCB (Process Control Block)

커널은 모든 프로세스를 `task_struct`라는 구조체로 관리한다. 이것이 PCB다.

```c
/* 리눅스 커널의 task_struct (주요 필드만) */
struct task_struct {
    volatile long        state;      /* -1: 실행 불가, 0: 실행 가능, >0: 정지 */
    pid_t                pid;        /* 프로세스 ID */
    pid_t                tgid;       /* 스레드 그룹 ID (멀티스레드 시 동일) */
    struct mm_struct     *mm;        /* 가상 주소 공간 (메모리 맵) */
    struct files_struct  *files;     /* 열린 파일 디스크립터 테이블 */
    struct signal_struct *signal;    /* 시그널 핸들러 */
    struct task_struct   *parent;    /* 부모 프로세스 */
    struct thread_struct thread;     /* CPU 레지스터 상태 */
    int                  prio;       /* 동적 우선순위 */
    u64                  utime;      /* 유저 모드 실행 시간 */
    u64                  stime;      /* 커널 모드 실행 시간 */
};
```

PCB가 저장하는 핵심 정보:

```
PCB (task_struct)
├── 프로세스 식별
│   ├── PID: 프로세스 ID
│   ├── PPID: 부모 프로세스 ID
│   └── TGID: 스레드 그룹 ID
│
├── 실행 상태
│   ├── R (Running / Runnable)        ← CPU에서 실행 중 or 실행 대기
│   ├── S (Interruptible Sleep)       ← I/O 대기, 시그널로 깨울 수 있음
│   ├── D (Uninterruptible Sleep)     ← 디스크 I/O 대기, SIGKILL도 안 먹힘
│   ├── T (Stopped)                   ← SIGSTOP으로 일시 정지
│   └── Z (Zombie)                    ← 종료됐지만 부모가 wait() 안 함
│
├── 메모리 정보
│   └── mm_struct: 페이지 테이블, VMA(Virtual Memory Area) 목록
│
├── CPU 레지스터 (컨텍스트 스위칭 시 저장/복원)
│   ├── 범용 레지스터 (rax, rbx, rcx ...)
│   ├── 명령어 포인터 (rip)
│   └── 스택 포인터 (rsp)
│
└── 파일 디스크립터 테이블
    └── 열린 파일, 소켓, 파이프 목록
```

### 프로세스 상태 전이

```
       fork()              스케줄러 선택
[생성] ──────→ [RUNNABLE] ──────────────→ [RUNNING]
                   ↑                           │
                   │     타임슬라이스 소진         │
                   └───────────────────────────┘
                                               │
                            I/O 요청            │
                   [SLEEPING/WAIT] ←───────────┘
                            │
                            │  I/O 완료 (인터럽트)
                            └──────→ [RUNNABLE]

   [RUNNING] ──exit()──→ [ZOMBIE] ──부모 wait()──→ [소멸]
```

`D` 상태(Uninterruptible Sleep)는 특별하다. 커널이 디스크 I/O 등 중요한 작업 중에 프로세스를 이 상태로 만든다. `SIGKILL`조차 처리하지 못한다. `kill -9`가 안 먹히는 프로세스는 대부분 이 상태다.

---

## 💻 실전 실험

### 실험 1: Java 프로세스의 가상 주소 공간 직접 확인

```bash
# Spring Boot 애플리케이션 실행 후 PID 확인
$ java -jar app.jar &
$ PID=$!

# 가상 주소 공간 매핑 전체 확인
$ cat /proc/$PID/maps
```

출력 예시 (일부):
```
주소 범위                    권한   오프셋   장치    inode   경로
55a1b0000000-55a1b0001000   r--p  00000000  08:01  123456  /usr/bin/java
55a1b0001000-55a1b0052000   r-xp  00001000  08:01  123456  /usr/bin/java  ← 코드
7f8a00000000-7f8b00000000   rw-p  00000000  00:00  0                       ← JVM Heap
7ffca4200000-7ffca4221000   rw-p  00000000  00:00  0       [stack]         ← 메인 스레드 스택
7ffca4400000-7ffca4402000   r--p  00000000  00:00  0       [vvar]
7ffca4402000-7ffca4404000   r-xp  00000000  00:00  0       [vdso]
```

권한 플래그 해석:
- `r`: 읽기 / `w`: 쓰기 / `x`: 실행 / `p`: 프라이빗(CoW) / `s`: 공유

```bash
# smaps로 영역별 실제 물리 메모리(RSS) 확인
$ grep -A 20 "\[heap\]" /proc/$PID/smaps
7f8a00000000-7f8b00000000 rw-p ...
Size:           262144 kB    ← 예약된 가상 크기
Rss:             51200 kB    ← 실제 물리 메모리에 올라온 크기
Pss:             51200 kB    ← 공유 메모리 비율 반영 크기
Private_Dirty:   51200 kB    ← 이 프로세스만 사용 중인 더티 페이지
```

```bash
# Java NMT로 영역별 상세 확인 (-XX:NativeMemoryTracking=summary 필요)
$ jcmd $PID VM.native_memory summary
Total: reserved=8192MB, committed=4500MB
- Java Heap (reserved=4096MB, committed=4096MB)
- Class       (reserved=1056MB, committed=52MB)   ← Metaspace
- Thread      (reserved=400MB,  committed=400MB)  ← 200 threads * 2MB
- Code        (reserved=256MB,  committed=40MB)   ← JIT 컴파일 코드
- GC          (reserved=200MB,  committed=120MB)
```

### 실험 2: 프로세스 상태 실시간 관찰

```bash
# PCB 핵심 정보 확인
$ cat /proc/$PID/status
Name:    java
State:   S (sleeping)
Tgid:    12345        ← 스레드 그룹 ID
Pid:     12345
PPid:    1            ← 부모 PID
Threads: 48           ← 현재 스레드 수
VmPeak:  8388608 kB   ← 최대 가상 메모리
VmRSS:   512000 kB    ← 실제 물리 메모리 (RSS)
VmStk:   136 kB       ← 메인 스레드 스택

# D 상태 프로세스 찾기 (I/O 블로킹 중)
$ ps aux | awk '$8 == "D" {print $0}'

# 해당 프로세스가 어떤 I/O를 기다리는지 확인
$ cat /proc/$PID/wchan
ext4_file_read_iter    ← 이 커널 함수에서 대기 중
```

### 실험 3: 스택 크기와 스레드 메모리

```bash
# 현재 스택 크기 한도 확인
$ ulimit -s
8192   # 기본값 8MB

# 스레드 200개의 스택 예약 크기 = 200 * 8MB = 1.6GB (가상 메모리)
# 실제 물리 메모리는 사용한 만큼만 Demand Paging으로 할당

# Java에서 스택 크기 조정
$ java -Xss512k -jar app.jar   # 스레드당 스택 512KB로 축소
```

---

## 📊 성능/비용 비교

| 항목 | 프로세스 | 스레드 |
|------|---------|-------|
| 가상 주소 공간 | 독립 (완전히 분리) | 공유 (같은 프로세스 내) |
| 스택 | 프로세스당 1개 | 스레드마다 독립 |
| 생성 비용 | 높음 (페이지 테이블 복사) | 낮음 (주소 공간 공유) |
| 격리 수준 | 완전 격리 | 부분 격리 (메모리 공유) |
| 하나가 크래시 시 | 다른 프로세스 영향 없음 | 전체 프로세스 영향 가능 |

**Java 프로세스 실제 메모리 분해 (`-Xmx4g` 설정 시)**:

```
영역                         크기 (예시)
─────────────────────────────────────────
Java Heap (Young + Old)     4,096 MB   ← -Xmx4g
Metaspace                     256 MB   ← 클래스 메타데이터
Code Cache                    256 MB   ← JIT 컴파일 코드
Thread Stack (200 threads)    400 MB   ← 200 * 2MB (기본)
GC 오버헤드 (G1 기준)          200 MB
JVM 내부 (Symbols 등)          100 MB
─────────────────────────────────────────
합계                        ~5,308 MB  ← -Xmx4g임에도 실제 5GB 이상!
```

---

## ⚖️ 트레이드오프

**가상 주소 공간의 장점과 비용**

가상 주소 공간은 프로세스 간 격리와 보호를 제공한다. 하지만 Page Table을 유지해야 하고, 주소 변환 시 TLB가 사용된다. TLB 미스가 잦으면 성능이 저하된다. 대용량 메모리를 사용하는 Redis나 JVM 같은 애플리케이션이 HugePages를 활용해 TLB 미스를 줄이는 이유가 여기에 있다.

**D 상태(Uninterruptible Sleep)의 트레이드오프**

커널이 I/O 도중 인터럽트를 허용하지 않는 이유는 데이터 일관성 때문이다. 디스크 쓰기 중에 프로세스가 강제 종료되면 파일이 손상될 수 있다. 대신 I/O 병목 시 `D` 상태 프로세스가 쌓여 시스템이 응답 불능처럼 보이는 단점이 있다. `D` 상태 프로세스가 많다면 디스크 I/O 병목을 의심해야 한다.

---

## 📌 핵심 정리

```
프로세스 = 실행 중인 프로그램 + 독립된 가상 주소 공간 + PCB

가상 주소 공간 구성 (낮은 → 높은 주소):
  [코드] → [데이터(.data/.bss)] → [힙 ↑] → ... → [↓ 스택]

PCB (task_struct):
  - 프로세스 상태 (R/S/D/T/Z)
  - 가상 주소 공간 정보 (mm_struct)
  - CPU 레지스터 (컨텍스트 스위칭 시 저장/복원)
  - 열린 파일 디스크립터

핵심 진단 명령어:
  /proc/<pid>/maps    → 가상 주소 공간 레이아웃
  /proc/<pid>/smaps   → 영역별 실제 물리 메모리 사용량 (RSS, Private_Dirty)
  /proc/<pid>/status  → PCB 핵심 정보 (상태, 스레드 수, 메모리)
  /proc/<pid>/wchan   → D 상태 시 대기 중인 커널 함수 확인

Java 메모리 진단 시:
  -Xmx만 보면 안 된다 → Metaspace + Stack + Native 합산 필요
  jcmd <pid> VM.native_memory summary 로 전체 확인
```

---

## 🤔 생각해볼 문제

**Q1.** `/proc/<pid>/maps`를 보면 같은 공유 라이브러리(`libc.so`)가 여러 프로세스에 나타난다. 이 라이브러리의 물리 메모리는 몇 벌 존재하는가?

<details>
<summary>해설 보기</summary>

**1벌**이다. 공유 라이브러리의 코드 세그먼트(읽기 전용)는 물리 메모리에 한 번만 올라가고, 여러 프로세스의 페이지 테이블이 동일한 물리 프레임을 가리킨다. 각 프로세스의 가상 주소는 다르지만 물리 주소는 같다. 이것이 공유 라이브러리가 메모리를 절약하는 원리다. `smaps`의 `Pss`(Proportional Set Size) 값은 공유 비율을 반영해 실제 점유 비용을 더 정확히 나타낸다.

</details>

**Q2.** Java 애플리케이션에서 스레드를 500개 만들었다. `-Xmx`를 늘리지 않아도 OOM이 발생할 수 있는가?

<details>
<summary>해설 보기</summary>

**그렇다.** 스레드 스택은 Java Heap이 아닌 Native Memory에서 할당된다. 스레드 500개 × 스택 1MB = 500MB가 Heap 외부에서 소비된다. `-Xmx`를 조절해도 이 공간은 별도로 필요하다. 스레드 수가 많다면 `-Xss`로 스택 크기를 줄이거나 스레드 풀 크기를 제한해야 한다. `jcmd <pid> VM.native_memory summary`의 `Thread` 항목으로 확인 가능하다.

</details>

**Q3.** `kill -9 <pid>`를 실행했는데 프로세스가 계속 살아있다. 어떻게 진단하는가?

<details>
<summary>해설 보기</summary>

`ps aux`로 상태 컬럼을 확인한다. `D`(Uninterruptible Sleep) 상태라면 `SIGKILL`도 처리할 수 없다. `cat /proc/<pid>/wchan`으로 어떤 커널 함수를 기다리는지 확인한다. 주로 NFS 마운트 응답 없음, 디스크 I/O 불량 등이 원인이다. I/O 문제를 해결하거나 시스템을 재시작하는 것 외에 방법이 없다.

</details>

---

**[홈으로 🏠](../README.md)** | **[다음: fork()와 exec() ➡️](./02-fork-exec-cow.md)**
