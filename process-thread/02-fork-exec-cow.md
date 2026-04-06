# 02. fork()와 exec() — Copy-On-Write의 실체

## 🎯 핵심 질문

- `fork()` 호출 직후 자식 프로세스는 부모의 메모리를 어떻게 "복사"하는가?
- Copy-On-Write(CoW)란 무엇이고, 언제 실제 복사가 일어나는가?
- Redis `BGSAVE`가 쓰기 요청을 받으면서도 일관된 스냅샷을 찍을 수 있는 원리는?
- `exec()`은 `fork()`와 어떻게 다른가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Redis `BGSAVE` 도중 메모리 사용량이 갑자기 치솟는다.** `BGSAVE`는 `fork()`를 호출해 자식 프로세스가 스냅샷을 생성하는 동안 부모는 계속 쓰기 요청을 받는다. 이때 부모가 기존 페이지를 수정하면 CoW로 인해 물리 메모리가 복사된다. 쓰기가 많은 Redis 서버에서 `BGSAVE` 중에 메모리가 최대 2배까지 증가할 수 있다. 이를 이해하지 못하면 OOM이 발생했을 때 원인을 찾을 수 없다.

**Docker 이미지의 레이어가 "CoW 파일 시스템"으로 동작한다.** OverlayFS는 CoW 원리를 파일 시스템 레벨로 적용한다. 컨테이너가 기반 이미지의 파일을 수정할 때만 해당 파일이 쓰기 레이어에 복사된다. `fork()`의 CoW와 동일한 원리다.

**`nginx -s reload`가 무중단으로 설정을 반영한다.** Nginx는 `fork()`로 새 워커를 만들고 기존 워커가 처리 중인 연결을 완료하면 종료시킨다. `fork()` 동작을 이해하면 무중단 배포 메커니즘이 보인다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 상황: Redis 서버 메모리 16GB 설정. BGSAVE 중 OOM 발생

# 운영자 판단:
# "Redis 데이터가 8GB니까 메모리 16GB면 충분하겠지"
# → BGSAVE 중 CoW로 인한 메모리 증가를 고려하지 않음
# → 쓰기 빈도가 높으면 8GB * 2 = 16GB가 필요할 수 있음

$ redis-cli info memory
used_memory: 8589934592       # 8GB
used_memory_rss: 9663676416   # 9GB (CoW로 약간 증가)

# BGSAVE 중에는?
# used_memory_rss가 12~15GB까지 치솟을 수 있음
# → OOM으로 Redis 프로세스 사망
```

또는:

```java
// "ProcessBuilder로 자식 프로세스를 만들면 부모 메모리가 2배 필요하겠지?"
// → CoW 덕분에 실제 복사는 쓰기 시점까지 지연됨
// → 자식이 읽기만 하면 추가 메모리가 거의 필요 없음
ProcessBuilder pb = new ProcessBuilder("python3", "script.py");
pb.start();
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# Redis BGSAVE CoW 메모리 증가 추적
$ redis-cli info persistence
rdb_last_bgsave_status: ok
rdb_changes_since_last_save: 15234   # 마지막 저장 후 변경 횟수

$ redis-cli info memory
used_memory: 8589934592
used_memory_rss: 9663676416
mem_allocator: jemalloc-5.3.0

# BGSAVE 중 CoW 메모리 증가 실시간 모니터링
$ watch -n 1 'redis-cli info memory | grep used_memory_rss'

# Redis 설정: BGSAVE 빈도를 줄여 CoW 노출 시간 단축
# redis.conf
save 900 1      # 15분에 1회 변경 시
save 300 10     # 5분에 10회 변경 시

# 서버 메모리 계획: 데이터 크기 * 1.5 ~ 2.0 여유 확보
```

---

## 🔬 내부 동작 원리

### fork() 동작 단계

```
[부모 프로세스]                          [커널]
      │
      │  fork() 호출
      ├──────────────────────────────────→ ① 새 task_struct 생성 (자식 PCB)
      │                                    ② 부모의 페이지 테이블 복사
      │                                       (가상 주소 매핑 구조를 복제)
      │                                    ③ 모든 페이지를 읽기 전용으로 표시
      │                                    ④ 자식 PID 반환 (부모에게)
      │                                    ⑤ 0 반환 (자식에게)
      │←──────────────────────────────────
      │
      ├── 부모: fork() 반환값 > 0 (자식 PID)
      │         계속 실행
      │
      └── 자식: fork() 반환값 = 0
                다른 코드 경로 실행
```

**핵심**: `fork()` 직후에는 물리 메모리를 복사하지 않는다. 부모와 자식 모두 같은 물리 페이지를 가리키는 페이지 테이블을 가지며, 두 테이블 모두 해당 페이지를 **읽기 전용**으로 표시한다.

### Copy-On-Write 메커니즘

```
fork() 직후:

[부모 페이지 테이블]    [물리 메모리]    [자식 페이지 테이블]
  가상주소 0x1000 ──→  [페이지 A] ←──  가상주소 0x1000
  (읽기 전용)           "data=42"         (읽기 전용)
  가상주소 0x2000 ──→  [페이지 B] ←──  가상주소 0x2000
  (읽기 전용)           "data=99"         (읽기 전용)

──── 부모가 페이지 A에 쓰기 시도 ────

[Page Fault 발생!]
커널: "읽기 전용 페이지에 쓰기 시도 → CoW 트리거"

① 물리 메모리에서 페이지 A를 복사 → [페이지 A']
② 부모의 페이지 테이블: 0x1000 → [페이지 A'] (쓰기 허용)
③ 자식의 페이지 테이블: 0x1000 → [페이지 A]  (여전히 원본)
④ 부모가 [페이지 A']에 수정 완료

결과:
[부모 페이지 테이블]    [물리 메모리]    [자식 페이지 테이블]
  가상주소 0x1000 ──→  [페이지 A']       가상주소 0x1000 ──→ [페이지 A]
  (쓰기 가능)           "data=100"        (읽기 전용)          "data=42"
  가상주소 0x2000 ──→  [페이지 B] ←──   가상주소 0x2000
```

쓰기가 발생한 페이지만 복사된다. 자식이 읽기만 한다면 추가 물리 메모리는 거의 필요 없다.

### Redis BGSAVE의 CoW 활용

```
[Redis 부모 프로세스]           [Redis 자식 프로세스]

BGSAVE 명령 수신
       │
       ├── fork() 호출
       │         ↓
       │   자식 프로세스 생성
       │   (모든 메모리 페이지: 읽기 전용 공유 상태)
       │
       │                           메모리 전체를 순회하며
       │                           RDB 형식으로 직렬화
       │                           ↓
클라이언트 요청 계속 처리             직렬화 중인 페이지: 원본 데이터
       │                           (부모가 수정해도 자식은 fork() 시점 값을 봄)
       │
쓰기 요청 발생
       │ → CoW 발생: 수정된 페이지만 복사
       │              (자식은 여전히 원본 페이지 읽기)
       │
       └── 자식이 .rdb 파일 저장 완료
           자식 종료

결론: 자식은 항상 fork() 시점의 일관된 스냅샷을 본다
      부모는 계속 쓰기를 처리하면서도 스냅샷 일관성이 보장된다
```

```bash
# BGSAVE 중 CoW로 인한 메모리 증가 확인
# rdb_current_bgsave_time_sec: BGSAVE 진행 시간
# used_memory_rss: CoW로 인한 실제 메모리 증가
$ redis-cli info all | grep -E "rdb_|used_memory"
```

### exec() — 주소 공간 교체

`fork()`와 달리 `exec()`은 현재 프로세스의 주소 공간을 완전히 새 프로그램으로 교체한다.

```
[fork() 후 자식]
       │
       │  exec("/bin/ls", ...)
       │
커널:  ① 현재 주소 공간의 모든 매핑 해제
       ② 새 실행 파일(ELF) 로드
       ③ 새로운 스택, 힙 설정
       ④ 새 프로그램의 main()부터 실행

결과: 같은 PID이지만 완전히 다른 프로그램이 실행됨
```

shell에서 명령어 실행 = `fork()` + `exec()`의 조합이다.

---

## 💻 실전 실험

### 실험 1: CoW 동작 직접 관찰

```c
/* cow_test.c */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

int main() {
    char *data = malloc(1024 * 1024 * 100);  // 100MB 할당
    memset(data, 'A', 1024 * 1024 * 100);    // 실제 물리 페이지 할당

    printf("fork() 전 RSS 확인: cat /proc/%d/status | grep VmRSS\n", getpid());
    sleep(5);

    pid_t pid = fork();

    if (pid == 0) {
        /* 자식: 읽기만 수행 */
        printf("[자식] fork() 후 - 읽기만 (CoW 발생 안 함)\n");
        printf("[자식] RSS 확인: cat /proc/%d/status | grep VmRSS\n", getpid());
        sleep(10);
        printf("[자식] 읽기 중: %c\n", data[0]);
        sleep(10);
        exit(0);
    } else {
        /* 부모: 쓰기 수행 → CoW 발생 */
        printf("[부모] fork() 후 - 쓰기 시작 (CoW 발생)\n");
        sleep(3);
        memset(data, 'B', 1024 * 1024 * 100);  // 100MB 전체 수정 → CoW
        printf("[부모] 쓰기 완료. RSS 확인: cat /proc/%d/status | grep VmRSS\n", getpid());
        sleep(20);
    }
    return 0;
}
```

```bash
$ gcc cow_test.c -o cow_test && ./cow_test &

# 별도 터미널에서 메모리 변화 관찰
$ while true; do
    echo "=== $(date) ==="
    ps aux | grep cow_test | grep -v grep | awk '{print "PID:"$2, "RSS:"$6"KB"}'
    sleep 1
  done
```

### 실험 2: Redis BGSAVE CoW 메모리 추적

```bash
# Docker로 Redis 실행
$ docker run -d --name redis-cow -p 6379:6379 redis:7

# 대량 데이터 삽입 (CoW 효과를 크게 하기 위해)
$ redis-cli --pipe << 'EOF'
$(for i in $(seq 1 100000); do echo "SET key:$i $(head -c 512 /dev/urandom | base64)"; done)
EOF

# 메모리 사용량 확인
$ redis-cli info memory | grep used_memory_rss

# 백그라운드에서 메모리 모니터링 시작
$ while true; do
    redis-cli info memory | grep -E "used_memory_rss|rdb_current_bgsave"
    sleep 0.5
  done &

# BGSAVE 실행
$ redis-cli BGSAVE

# CoW로 인해 used_memory_rss가 증가하는 것을 관찰
```

### 실험 3: strace로 fork/exec 시스템 콜 추적

```bash
# shell이 명령어를 실행할 때 fork+exec 확인
$ strace -e trace=clone,execve -f bash -c "ls /tmp" 2>&1 | head -20

# 출력 예시:
# clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD,
#        child_tidptr=0x7f...280) = 12345   ← fork() (리눅스에서는 clone()으로 구현)
# [pid 12345] execve("/bin/ls", ["ls", "/tmp"], ...) = 0  ← exec()
```

---

## 📊 성능/비용 비교

**fork() 비용: 메모리가 클수록 증가하는 이유**

```
메모리 크기와 fork() 지연 관계:

fork()가 하는 일:
  1. task_struct 생성         → O(1), 빠름
  2. 페이지 테이블 복사        → O(메모리 크기), 느려질 수 있음
  3. 모든 페이지 읽기 전용 표시 → O(페이지 수)

Redis 예시:
  데이터 8GB = 약 2,097,152 페이지 (4KB 기준)
  fork() 시 이 페이지들의 페이지 테이블 엔트리를 모두 읽기 전용으로 설정
  → 수십 ms ~ 수백 ms 소요 가능 (이 시간 동안 Redis 응답 지연)

$ redis-cli info stats | grep rdb_last_bgsave_time
rdb_last_bgsave_time_sec: 45  ← 마지막 BGSAVE 소요 시간
```

**CoW 발생 시 물리 메모리 증가량**:

```
시나리오: Redis 8GB, BGSAVE 진행 중

쓰기 비율 10% (적은 쓰기):
  CoW로 복사된 페이지: 0.8GB
  총 물리 메모리: 8 + 0.8 = 8.8GB

쓰기 비율 50% (중간):
  CoW로 복사된 페이지: 4GB
  총 물리 메모리: 8 + 4 = 12GB

쓰기 비율 100% (전체 페이지 수정):
  CoW로 복사된 페이지: 8GB
  총 물리 메모리: 8 + 8 = 16GB ← 최악의 경우
```

---

## ⚖️ 트레이드오프

**CoW의 장점**:
- `fork()` 속도가 매우 빠르다 (실제 복사 없음)
- 자식이 읽기만 하면 추가 메모리가 거의 필요 없다
- Redis BGSAVE처럼 일관된 스냅샷을 저렴하게 얻을 수 있다

**CoW의 단점**:
- 쓰기 발생 시 Page Fault가 일어나 성능이 저하된다
- 쓰기가 많은 환경에서 BGSAVE 중 메모리 사용량이 급증한다
- THP(Transparent HugePages)가 활성화된 경우 CoW 단위가 2MB가 되어 작은 쓰기에도 2MB 복사 발생 → Redis가 THP를 비활성화 권장하는 이유

```bash
# Redis THP 비활성화 확인
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never   ← madvise 또는 never 권장

# Redis 시작 시 경고 메시지
# "WARNING you have Transparent Huge Pages (THP) support enabled in your kernel.
#  This will create latency and memory usage issues with Redis."
```

---

## 📌 핵심 정리

```
fork():
  1. task_struct(PCB) 복사
  2. 페이지 테이블 복사 (물리 메모리 복사 아님!)
  3. 모든 페이지 읽기 전용으로 표시
  4. 자식: 반환값 0 / 부모: 반환값 = 자식 PID

Copy-On-Write:
  - 쓰기 발생 → Page Fault → 커널이 해당 페이지만 복사
  - 쓰기 없는 페이지는 부모/자식이 영원히 공유

Redis BGSAVE:
  fork() 시점의 페이지 테이블 스냅샷 = 일관된 데이터 뷰
  → 자식은 항상 fork() 순간의 데이터를 직렬화
  → 부모는 CoW로 계속 쓰기 처리 가능
  → 주의: 쓰기 빈도가 높으면 메모리 2배 필요 가능

exec():
  현재 주소 공간을 완전히 새 프로그램으로 교체
  fork() 후 exec() = 새 프로세스로 다른 프로그램 실행 (shell의 동작)

핵심 진단:
  BGSAVE 중 메모리 급증 → CoW 발생량 = 쓰기 비율 * 데이터 크기
  THP 비활성화 확인: /sys/kernel/mm/transparent_hugepage/enabled
```

---

## 🤔 생각해볼 문제

**Q1.** Redis 서버에 데이터가 10GB 있고, `BGSAVE` 실행 중 초당 500MB의 쓰기가 발생한다. BGSAVE에 30초가 걸린다면 CoW로 인해 얼마의 추가 메모리가 필요한가? (단순 계산)

<details>
<summary>해설 보기</summary>

이론적 최대치: 500MB/s × 30s = 15GB. 단, 동일한 페이지를 여러 번 수정하면 CoW는 한 번만 발생하므로 실제로는 10GB(전체 데이터)를 초과할 수 없다. 현실적으로 쓰기가 고르게 분산된다면 10GB * (수정된 페이지 비율)의 추가 메모리가 필요하다. 이 때문에 Redis 서버는 데이터 크기의 1.5~2배 메모리를 준비하는 것이 권장된다.

</details>

**Q2.** `fork()` 후 자식 프로세스가 즉시 `exec()`을 호출한다. CoW 복사가 발생하는가?

<details>
<summary>해설 보기</summary>

**거의 발생하지 않는다.** `exec()`은 주소 공간 전체를 교체하기 때문에 기존 페이지에 쓸 필요가 없다. 이것이 `fork()` + `exec()` 패턴이 효율적인 이유다. 단, `exec()` 호출 전 스택에 지역 변수 수정이나 파일 디스크립터 설정 같은 소량의 쓰기는 발생할 수 있다. 이 점을 최적화한 것이 Linux의 `posix_spawn()` 이다.

</details>

**Q3.** Docker 이미지 레이어가 CoW를 사용한다. 컨테이너 안에서 1GB 파일을 수정하면 실제로 어떤 일이 일어나는가?

<details>
<summary>해설 보기</summary>

OverlayFS의 CoW 동작: 파일이 수정되면 하위 레이어(읽기 전용 이미지)에서 상위 레이어(쓰기 가능 컨테이너 레이어)로 파일 전체가 복사(Copy-Up)된 후 수정된다. 1GB 파일의 1바이트만 수정해도 1GB 전체가 컨테이너 레이어에 복사된다. 이것이 컨테이너 레이어가 예상보다 크게 증가하는 이유다. `docker diff <container>`로 변경된 파일 목록을 확인할 수 있다.

</details>

---

**[⬅️ 이전: 프로세스란 무엇인가](./01-process-address-space.md)** | **[홈으로 🏠](../README.md)** | **[다음: 스레드 모델 ➡️](./03-thread-model.md)**
