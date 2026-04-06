# 01. 파일 디스크립터와 시스템 콜 — 커널의 관문

## 🎯 핵심 질문

- 파일 디스크립터(fd)는 무엇이고, 커널 내부의 어떤 구조를 가리키는가?
- `read()` 시스템 콜 한 번 호출 시 커널에서 실제로 무슨 일이 벌어지는가?
- 유저 모드 ↔ 커널 모드 전환 비용은 얼마이며, 왜 시스템 콜을 줄여야 하는가?
- `strace`로 애플리케이션의 I/O 패턴을 어떻게 분석하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Netty가 "Zero-copy"라고 말할 때 실제로 무엇을 줄이는가?** 일반 `read()` + `write()` 경로는 커널 버퍼 → 유저 버퍼 → 커널 버퍼로 두 번 복사된다. `sendfile()` 시스템 콜은 이 복사를 없애 커널 버퍼에서 NIC로 직접 전달한다. 이를 이해하려면 시스템 콜이 커널에서 어떻게 동작하는지 알아야 한다.

**Java NIO의 `FileChannel.read()`가 `InputStream.read()`보다 빠른 이유는?** `FileChannel`은 커널 버퍼를 Direct Buffer(Native Memory)에 직접 매핑하여 JVM 힙으로의 추가 복사를 줄인다. 시스템 콜 흐름을 알면 이 차이가 보인다.

**`strace -c -p <pid>`를 실행했더니 `read`가 99%를 차지한다.** 어떤 fd를 읽고 있는지, 반환 값이 `EAGAIN`(데이터 없음)인지 실제 데이터인지에 따라 진단 방향이 완전히 달라진다. 시스템 콜 레벨에서 애플리케이션을 보는 눈이 생기면 블랙박스가 사라진다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```java
// 실수 1: 파일을 1바이트씩 읽기 (시스템 콜 폭발)
FileInputStream fis = new FileInputStream("large.log");
int b;
while ((b = fis.read()) != -1) {  // ← 1바이트마다 시스템 콜!
    process(b);
}
// 100MB 파일 = 약 1억 번의 read() 시스템 콜
// → 엄청난 유저↔커널 전환 오버헤드

// 올바른 접근: 버퍼 사용
byte[] buf = new byte[8192];
int n;
while ((n = fis.read(buf)) != -1) {  // 8KB씩 읽기 = 약 12,800번 시스템 콜
    process(buf, n);
}
```

```bash
# 실수 2: 열린 파일 디스크립터 누수 인지 못 함
$ lsof -p $PID | wc -l
10482   ← 파일 디스크립터 1만 개!
# "그냥 많은 것 아닌가?" → fd 한도 초과 시 "Too many open files" 오류
# close()를 안 한 소켓, 파일이 누적된 것
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# strace로 시스템 콜 통계 수집 (30초)
$ strace -c -p $PID sleep 30

# 출력 예시:
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- --------
 45.23    0.023456          23      1024           epoll_wait
 30.12    0.015634          15      1042           read
 15.45    0.008012           8      1001           write
  5.20    0.002700           2      1350           recvfrom
# → epoll_wait이 45%: 이벤트 루프 정상 동작
# → read 30%: 데이터 수신 처리
# 문제 있는 경우: futex(락 경합), poll/select(O(N) 스캔) 높으면 조사 필요

# 특정 시스템 콜만 추적 (I/O 관련)
$ strace -e trace=read,write,open,close,socket,accept,connect \
  -p $PID -f 2>&1 | head -50

# fd 목록과 연결된 파일/소켓 확인
$ ls -la /proc/$PID/fd/ | head -20
lrwxrwxrwx 1 app app 64 fd/0  -> /dev/null
lrwxrwxrwx 1 app app 64 fd/1  -> /dev/null
lrwxrwxrwx 1 app app 64 fd/4  -> socket:[123456]
lrwxrwxrwx 1 app app 64 fd/5  -> /var/log/app.log
```

---

## 🔬 내부 동작 원리

### 파일 디스크립터의 정체

```
프로세스가 파일을 열면 세 개의 테이블이 연결된다:

[프로세스]
  task_struct
  └── files_struct (파일 디스크립터 테이블)
       ├── fd[0] → [파일 테이블 엔트리]
       │              ├── 파일 오프셋 (현재 읽기 위치)
       │              ├── 접근 모드 (O_RDONLY 등)
       │              └── → [inode]
       │                      ├── 파일 크기, 권한, 소유자
       │                      └── → 디스크 블록 주소
       ├── fd[1] → [파일 테이블 엔트리] (stdout)
       ├── fd[2] → [파일 테이블 엔트리] (stderr)
       └── fd[3] → [파일 테이블 엔트리] (열린 파일)

fd(파일 디스크립터):
  단순한 정수 인덱스 (0, 1, 2, 3 ...)
  이 인덱스 → 파일 테이블 엔트리 → inode → 실제 파일/소켓/파이프

소켓도 fd다:
  accept() → 새 fd 반환 (연결된 소켓을 가리킴)
  recv(fd, buf, len, 0) → 소켓 fd로 데이터 수신
  파일과 소켓이 동일한 read()/write() 인터페이스를 공유하는 이유
```

### 시스템 콜 — 유저↔커널 전환

```
시스템 콜이 없다면:
  모든 프로세스가 하드웨어에 직접 접근 → 격리 불가
  → 커널이 하드웨어 접근의 유일한 관문 = 시스템 콜

유저 모드 → 커널 모드 전환 과정 (x86-64):

[유저 공간]
  read(fd, buf, count)  ← C 라이브러리 호출
       │
       │ syscall 명령어 실행 (또는 int 0x80)
       │ rax = 0 (read의 시스템 콜 번호)
       │
       ▼ [CPU 모드 전환: Ring 3 → Ring 0]
[커널 공간]
  1. 현재 CPU 레지스터 저장 (유저 스택)
  2. 커널 스택으로 전환
  3. 시스템 콜 핸들러 진입 (sys_read)
  4. fd 유효성 검사, 권한 확인
  5. 파일/소켓 타입에 따른 읽기 수행
  6. 결과를 유저 버퍼에 복사 (copy_to_user)
  7. 반환값을 레지스터에 설정
       │
       ▼ [CPU 모드 전환: Ring 0 → Ring 3]
[유저 공간]
  반환값 처리

전환 비용: ~100ns~1μs (TLB 부분 무효화, 레지스터 저장/복원)
→ 시스템 콜 1억 번 = 100ms~1s 오버헤드만으로 낭비
```

### read() 시스템 콜 내부 동작

```
read(sockfd, buf, 4096) 호출 시:

[커널 sys_read]
       │
       ▼
  fd → 파일 테이블 → inode → 파일 오퍼레이션
       │
       ├── 일반 파일이면:
       │   Page Cache 확인 → 히트: 복사 / 미스: 디스크 읽기
       │
       └── 소켓이면:
           수신 버퍼(sk_buff) 확인
           │
           ├── 데이터 있음: sk_buff → 유저 buf 복사 (copy_to_user)
           │               반환값: 복사된 바이트 수
           │
           ├── 데이터 없음 + Blocking:
           │   프로세스를 소켓 대기 큐에 추가
           │   → 스케줄러가 다른 프로세스 실행
           │   → 데이터 도착 시 인터럽트 → Wake-up → 복사
           │
           └── 데이터 없음 + O_NONBLOCK:
               즉시 EAGAIN(-11) 반환
               → 프로세스 블로킹 없음

커널 수신 버퍼(sk_buff) 구조:
  NIC → DMA → sk_buff (커널 메모리)
  sk_buff → copy_to_user() → 애플리케이션 버퍼
  ↑ 이 복사가 "Zero-copy"가 없애려는 복사
```

### /proc/pid/fd 해석

```bash
$ ls -la /proc/$(pgrep java)/fd
lrwxrwxrwx fd/0  -> /dev/null          ← stdin (닫힘)
lrwxrwxrwx fd/1  -> /dev/null          ← stdout (닫힘)
lrwxrwxrwx fd/2  -> /dev/null          ← stderr (닫힘)
lrwxrwxrwx fd/3  -> socket:[12345]     ← TCP 소켓
lrwxrwxrwx fd/4  -> /app/app.jar       ← JAR 파일 (mmap으로 로딩됨)
lrwxrwxrwx fd/5  -> anon_inode:eventpoll ← epoll 인스턴스
lrwxrwxrwx fd/6  -> pipe:[67890]       ← 파이프
lrwxrwxrwx fd/7  -> /var/log/app.log   ← 로그 파일

# fd 한도 확인
$ ulimit -n
65536   ← 소프트 한도
$ cat /proc/sys/fs/file-max
9223372036854775807  ← 시스템 전체 한도
```

---

## 💻 실전 실험

### 실험 1: strace로 Java 애플리케이션 I/O 패턴 분석

```bash
# Spring Boot 실행 후 strace 연결
$ java -jar app.jar &
$ PID=$!
$ sleep 5  # 시작 완료 대기

# 30초간 시스템 콜 통계
$ strace -c -p $PID sleep 30 2>&1

# 특정 I/O 콜만 추적 (읽기 쓰기 패턴)
$ strace -e trace=read,write,recvfrom,sendto \
  -T -p $PID 2>&1 | head -30
# -T: 각 콜 소요 시간 표시
# 출력 예:
# read(5, "\x00\x00\x00\x1a...", 8192) = 512 <0.000021>  ← 0.021ms
# write(6, "HTTP/1.1 200 OK\r\n...", 256) = 256 <0.000008>

# 어떤 fd에서 어떤 데이터가 오는지 확인
$ strace -e trace=read -p $PID -f 2>&1 | \
  awk '{match($0, /read\(([0-9]+)/, arr); print "fd="arr[1]}' | \
  sort | uniq -c | sort -rn | head -5
# 가장 많이 읽는 fd 순위
```

### 실험 2: fd 누수 탐지

```bash
# 파일 디스크립터 수 모니터링
$ watch -n 1 "ls /proc/$PID/fd | wc -l"

# fd 누수 의심 시 종류별 집계
$ ls -la /proc/$PID/fd | awk '{print $NF}' | \
  sed 's/socket:\[.*/socket/' | \
  sed 's/\/.*\//file:\//' | \
  sort | uniq -c | sort -rn
# 출력 예:
#  450 socket           ← TCP 소켓 450개 (정상?)
#  200 file:/var/log/   ← 로그 파일 200개 (누수!)
#    1 anon_inode:eventpoll

# 소켓 상태 교차 확인
$ ss -tanp | grep $PID | awk '{print $1}' | sort | uniq -c
# ESTABLISHED 450   ← 소켓 수와 일치하면 정상
```

### 실험 3: 시스템 콜 횟수와 성능 영향 측정

```c
/* syscall_cost.c */
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <time.h>
#include <string.h>

#define N 1000000

long now_ns() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000000000L + ts.tv_nsec;
}

int main() {
    int fd = open("/dev/null", O_WRONLY);
    char buf[1] = {0};
    char buf8k[8192];
    memset(buf8k, 0, 8192);

    /* 1바이트씩 N번 write (N번 시스템 콜) */
    long start = now_ns();
    for (int i = 0; i < N; i++) write(fd, buf, 1);
    printf("1바이트 * %d회: %ldms\n", N, (now_ns()-start)/1000000);

    /* 8KB씩 N/8192번 write */
    int calls = N / 8192;
    start = now_ns();
    for (int i = 0; i < calls; i++) write(fd, buf8k, 8192);
    printf("8KB * %d회: %ldms\n", calls, (now_ns()-start)/1000000);

    close(fd);
    return 0;
}
```

```bash
$ gcc syscall_cost.c -o syscall_cost && ./syscall_cost
# 예상 출력:
# 1바이트 * 1000000회: 890ms   ← 시스템 콜 100만 번
# 8KB * 122회: 1ms             ← 시스템 콜 122번
# → 버퍼 크기의 중요성이 수치로 확인됨
```

---

## 📊 성능/비용 비교

| 항목 | 유저 모드 함수 호출 | 시스템 콜 |
|------|-----------------|---------|
| CPU 모드 전환 | 없음 | Ring 3 → Ring 0 → Ring 3 |
| 비용 | ~1ns | ~100ns~1μs |
| TLB 영향 | 없음 | 부분 무효화 가능 |
| 적합한 사용 | 계산, 메모리 조작 | I/O, 프로세스 관리 |

**시스템 콜별 평균 비용 (현대 x86-64)**:

```
getpid():          ~20ns   (최소 비용, 캐싱됨)
gettimeofday():    ~25ns   (vDSO로 최적화)
read() 소형:       ~200ns  (캐시 히트 시)
read() 대형:       ~수μs   (데이터 크기 비례)
socket():         ~500ns
accept():         ~1μs
epoll_wait():     ~수μs   (이벤트 수 비례)
```

**버퍼 크기별 처리량 비교**:

```
파일 읽기 100MB:
  버퍼 1B:   시스템 콜 1억 번,  처리 시간 ~90초
  버퍼 4KB:  시스템 콜 25600번, 처리 시간 ~0.05초
  버퍼 64KB: 시스템 콜 1600번,  처리 시간 ~0.04초
  버퍼 1MB:  시스템 콜 100번,   처리 시간 ~0.04초
  ↑ 4KB 이상이면 시스템 콜 횟수 감소 효과가 포화됨
```

---

## ⚖️ 트레이드오프

**시스템 콜 배칭 vs 레이턴시**

시스템 콜을 줄이려면 데이터를 모아서 한 번에 보내는 배칭이 효과적이다. TCP의 Nagle 알고리즘이 이를 자동으로 한다. 하지만 배칭은 레이턴시를 높인다. 소량의 데이터를 자주 보내는 Redis 프로토콜에서 `TCP_NODELAY`로 Nagle을 비활성화하는 이유가 여기에 있다.

**fd 테이블 크기와 성능**

fd 테이블은 동적으로 성장하지만, 많은 fd를 열면 커널이 관리해야 할 구조체가 늘어난다. `select()`는 `FD_SETSIZE=1024` 제한이 있고, `poll()`은 fd 수에 비례한 O(N) 오버헤드가 있다. `epoll`은 이 문제를 해결했지만, fd 자체가 많으면 메모리와 관리 비용이 증가한다.

---

## 📌 핵심 정리

```
파일 디스크립터:
  정수 인덱스 → 파일 테이블 엔트리 → inode
  파일, 소켓, 파이프, epoll 인스턴스 모두 fd로 표현
  표준: 0(stdin), 1(stdout), 2(stderr)

시스템 콜:
  유저 → 커널 모드 전환 (syscall 명령어)
  비용: ~100ns~1μs
  최소화 전략: 버퍼링, 배칭, io_uring(최신)

read() 흐름:
  소켓: NIC → DMA → sk_buff → copy_to_user → 앱 버퍼
  파일: 디스크 → Page Cache → copy_to_user → 앱 버퍼

핵심 진단 명령어:
  strace -c -p <pid>        → 시스템 콜 통계 (30초)
  strace -e read,write -T   → 각 콜 소요 시간
  ls /proc/<pid>/fd         → 열린 fd 목록
  lsof -p <pid>             → fd + 파일/소켓 상세
  ulimit -n                 → fd 소프트 한도
  /proc/sys/fs/file-max     → 시스템 전체 한도
```

---

## 🤔 생각해볼 문제

**Q1.** `fork()` 후 부모와 자식이 같은 fd(예: 소켓)를 공유한다. 자식이 `read()`로 소켓에서 읽으면 부모는 같은 데이터를 읽을 수 있는가?

<details>
<summary>해설 보기</summary>

**없다.** `read()`는 소켓 수신 버퍼에서 데이터를 소비한다. 자식이 읽으면 해당 데이터는 버퍼에서 제거된다. 부모가 같은 fd로 `read()`를 호출해도 이미 소비된 데이터는 없다. `fork()` 후 일반적으로 부모나 자식 중 하나만 해당 fd를 사용하고, 나머지는 `close()`로 닫아야 한다. Nginx의 멀티 프로세스 모델에서 각 워커가 독립적으로 Accept를 처리하는 이유다.

</details>

**Q2.** Java `FileInputStream.read(buf, 0, 8192)`는 시스템 콜을 몇 번 호출하는가?

<details>
<summary>해설 보기</summary>

**1번**이다. Java의 `read(byte[], int, int)` 메서드는 내부적으로 `read()` 시스템 콜 1번을 호출하고 최대 8192바이트를 읽어온다. 단, 반환값이 8192보다 작을 수 있다(소켓의 경우 가용 데이터만큼만 반환). 반면 `read()` (인자 없는 버전, 1바이트 읽기)는 매번 시스템 콜 1번이다. `BufferedInputStream`은 내부 버퍼(기본 8KB)를 채울 때만 시스템 콜을 하고, 이후 버퍼에서 제공한다.

</details>

**Q3.** `strace`를 운영 서버에서 사용할 때 성능 영향은 어느 정도인가?

<details>
<summary>해설 보기</summary>

`strace`는 `ptrace` 시스템 콜을 기반으로 동작한다. 모든 시스템 콜마다 커널이 `strace` 프로세스에 알림을 보내고 재개하는 과정이 발생해 **성능이 40~70% 저하**될 수 있다. 운영 환경에서 `-c` 옵션(통계만 수집)은 그나마 낫지만 여전히 부담이다. 더 가벼운 대안으로 `perf trace`(eBPF 기반)가 있으며, 오버헤드가 5% 이하다. 운영 환경에서는 `strace` 대신 `perf` 또는 `BPF` 도구 사용을 권장한다.

</details>

---

**[홈으로 🏠](../README.md)** | **[다음: Blocking I/O ➡️](./02-blocking-io.md)**
