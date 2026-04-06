# 01. 가상 메모리 — Page Table과 TLB

## 🎯 핵심 질문

- 모든 프로세스가 독립된 가상 주소 공간을 가지는 이유는 무엇인가?
- Page Table은 가상 주소를 물리 주소로 어떻게 변환하는가?
- TLB는 무엇이고, TLB 미스가 성능에 미치는 영향은?
- HugePages가 Redis와 JVM 성능에 왜 중요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Redis 서버에 `transparent_hugepage=always`가 설정돼 있으면 레이턴시가 불규칙하게 튄다.** THP(Transparent HugePages)가 활성화되면 CoW 시 2MB 단위로 복사가 발생해 `BGSAVE` 중 레이턴시 스파이크가 생긴다. Redis 공식 문서가 THP 비활성화를 강력히 권장하는 이유가 여기에 있다.

**JVM이 `-Xmx4g`인데 실제 물리 메모리 사용량이 더 적다.** Demand Paging 때문이다. JVM이 4GB를 예약(가상 메모리)해도 실제로 접근한 페이지만 물리 메모리에 올라온다. `vmstat`의 RSS와 VSZ의 차이가 이것이다.

**MySQL `innodb_buffer_pool_size`를 물리 메모리의 80%로 설정하라는 권장사항의 근거는?** InnoDB Buffer Pool은 가상 주소 공간에 할당되고, 페이지가 접근될 때마다 물리 메모리에 매핑된다. OS의 Page Cache와 이중 캐싱이 발생하지 않도록 `O_DIRECT`와 함께 설정하는 원리를 이해해야 올바른 크기 계산이 가능하다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: VSZ(가상 크기)와 RSS(실제 물리 메모리)를 혼동
$ ps aux | grep java
USER  PID  %CPU %MEM    VSZ    RSS   COMMAND
app   123   2.0  5.0  8388608 512000 java -jar app.jar
#                     ↑ 8GB 가상   ↑ 512MB 실제
# "메모리 8GB 쓰고 있네" → 잘못된 판단
# 실제 물리 메모리는 RSS 기준 512MB

# 실수 2: Redis THP 경고 무시
# Redis 시작 시:
# "WARNING you have Transparent Huge Pages (THP) support enabled"
# → "경고니까 무시해도 되겠지"
# → BGSAVE 중 레이턴시 수백 ms 스파이크 발생
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# VSZ vs RSS 올바른 해석
$ cat /proc/$PID/status | grep -E "VmPeak|VmRSS|VmSize"
VmPeak:  8388608 kB   ← 최대 가상 메모리 사용량
VmSize:  8192000 kB   ← 현재 가상 메모리 예약량
VmRSS:    512000 kB   ← 실제 물리 메모리 (이게 진짜 사용량)

# THP 비활성화 (Redis 권장)
$ echo never > /sys/kernel/mm/transparent_hugepage/enabled
$ echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 영구 설정 (/etc/rc.local 또는 systemd)
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]   ← never가 선택된 상태

# HugePages 명시적 설정 (JVM 성능 향상)
$ java -XX:+UseHugeTLBFS -Xmx4g -jar app.jar
# 또는
$ java -XX:+UseLargePages -Xmx4g -jar app.jar
```

---

## 🔬 내부 동작 원리

### 가상 메모리가 존재하는 이유

물리 메모리를 직접 쓰면 세 가지 문제가 생긴다.

```
문제 1: 격리 불가
  프로세스 A가 주소 0x1000에 쓰면 → 프로세스 B의 0x1000 데이터 덮어씀

문제 2: 단편화
  프로세스들이 메모리를 다른 크기로 요청 → 조각난 빈 공간 낭비

문제 3: 크기 제한
  물리 메모리보다 큰 프로그램 실행 불가

해결: 가상 주소 공간
  각 프로세스에게 독립된 가상 주소 공간 부여
  → 격리 보장 (다른 프로세스 주소에 접근 불가)
  → 연속된 가상 주소가 불연속 물리 주소에 매핑 가능
  → Demand Paging으로 실제 필요한 부분만 물리 메모리 사용
```

### Page Table 구조 (4단계, x86-64)

64비트 리눅스는 4단계(또는 5단계) Page Table을 사용한다.

```
가상 주소 (48비트 유효 주소):
┌──────┬──────┬──────┬──────┬────────────────┐
│ PGD  │ PUD  │ PMD  │ PTE  │  페이지 내 오프셋  │
│ 9비트 │ 9비트 │ 9비트 │ 9비트 │    12비트        │
└──────┴──────┴──────┴──────┴────────────────┘
  ↓       ↓       ↓       ↓
Page  Page  Page  Page
Global Upper  Middle Table
Directory Dir  Dir  Entry
                           ↓
                      물리 페이지 주소 (4KB 단위)

변환 과정:
1. CR3 레지스터 → PGD 테이블 주소
2. 가상주소[47:39] → PGD 인덱스 → PUD 주소
3. 가상주소[38:30] → PUD 인덱스 → PMD 주소
4. 가상주소[29:21] → PMD 인덱스 → PTE 주소
5. 가상주소[20:12] → PTE 인덱스 → 물리 페이지 주소
6. 가상주소[11:0]  → 페이지 내 오프셋 → 최종 물리 주소

메모리 접근 4번 필요 → TLB 없으면 엄청나게 느림
```

### TLB (Translation Lookaside Buffer)

```
TLB = CPU 내부의 주소 변환 캐시
  용량: 보통 64~1024 엔트리
  속도: ~1 사이클 (L1 캐시와 동등)

TLB 히트 (가장 일반적인 경우):
  가상 주소 → TLB 조회 → 물리 주소 즉시 반환
  비용: ~1 사이클

TLB 미스 (TLB에 없을 때):
  가상 주소 → TLB 미스
  → CPU가 Page Table Walk 수행 (메모리 4번 접근)
  → 물리 주소 획득 → TLB에 캐싱
  비용: ~수십~수백 사이클

TLB 플러시 발생 시점:
  - 다른 프로세스로 컨텍스트 스위칭 (CR3 변경)
  - munmap()으로 메모리 해제
  - mprotect()로 권한 변경
  → 플러시 후 모든 메모리 접근에서 TLB 미스 발생
```

### HugePages와 TLB 효율

```
기본 페이지 크기: 4KB
HugePages 크기: 2MB (또는 1GB)

TLB 커버리지 비교 (TLB 512 엔트리 기준):
  4KB 페이지:   512 * 4KB  =   2MB 커버
  2MB HugePages: 512 * 2MB = 1024MB 커버

JVM Heap 4GB 접근 패턴:
  4KB 페이지: TLB로 2MB만 커버 → 나머지 접근 시 TLB 미스 폭발
  2MB HugePage: TLB로 1GB 커버 → TLB 미스 대폭 감소

Redis + HugePages:
  BUT: CoW 발생 시 2MB 단위로 복사
  → BGSAVE 중 1바이트 수정해도 2MB 복사
  → 메모리 증가량 폭발
  → THP 비활성화 후 명시적 HugePages 사용 권장 (redis.conf: hugepages yes)
```

### /proc로 페이지 테이블 상태 확인

```bash
# 프로세스의 가상 메모리 영역 (VMA) 목록
$ cat /proc/$PID/maps
55a1b0000000-55a1b0400000 r-xp 00000000 08:01 12345  /usr/bin/java  ← 코드
7f8a00000000-7f8b00000000 rw-p 00000000 00:00 0                     ← JVM Heap
7ffca4200000-7ffca4221000 rw-p 00000000 00:00 0      [stack]

# 각 영역의 상세 물리 메모리 사용량
$ cat /proc/$PID/smaps | grep -A 15 "\[heap\]"
7f8a00000000-7f8b00000000 rw-p ...
Size:           1048576 kB   ← 예약된 가상 크기 (1GB)
Rss:             204800 kB   ← 실제 물리 메모리 (200MB)
Pss:             204800 kB   ← 공유 고려한 비례 크기
Shared_Clean:         0 kB
Shared_Dirty:         0 kB
Private_Clean:        0 kB
Private_Dirty:   204800 kB   ← 수정된 프라이빗 페이지
Referenced:      204800 kB
Anonymous:       204800 kB
AnonHugePages:   163840 kB   ← THP로 할당된 크기
```

---

## 💻 실전 실험

### 실험 1: Demand Paging 동작 확인

```c
/* demand_paging.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main() {
    size_t size = 1024 * 1024 * 512; // 512MB
    char *mem = malloc(size);

    printf("malloc() 직후 RSS 확인: cat /proc/%d/status | grep VmRSS\n", getpid());
    sleep(5);  // ← 이 시점: 가상 메모리만 예약, 물리 메모리 할당 안 됨

    printf("페이지 접근 시작...\n");
    // 4KB(페이지 크기)마다 한 번씩 접근 → Page Fault 발생
    for (size_t i = 0; i < size; i += 4096) {
        mem[i] = 1;
    }

    printf("접근 완료 후 RSS 확인: cat /proc/%d/status | grep VmRSS\n", getpid());
    sleep(10);

    free(mem);
    return 0;
}
```

```bash
$ gcc demand_paging.c -o demand_paging
$ ./demand_paging &

# 별도 터미널에서 RSS 변화 관찰
$ watch -n 0.5 "cat /proc/$(pgrep demand_paging)/status | grep -E 'VmRSS|VmSize'"

# malloc() 직후:
# VmSize: 524288 kB  (512MB 가상 예약)
# VmRSS:    1024 kB  (1MB 실제 사용 - 스택 등)

# 접근 완료 후:
# VmSize: 524288 kB  (동일)
# VmRSS:  524288 kB  (512MB 실제 할당)
```

### 실험 2: Page Fault 횟수 측정

```bash
# /usr/bin/time으로 Page Fault 통계 확인
$ /usr/bin/time -v ls /tmp 2>&1 | grep -E "Major|Minor"
Major (requiring I/O) page faults: 0    ← 디스크 I/O 필요한 Page Fault
Minor (reclaiming a frame) page faults: 142  ← 물리 페이지 매핑만 필요

# Java 애플리케이션의 Page Fault 확인
$ /usr/bin/time -v java -jar app.jar 2>&1 | grep "page faults"
```

### 실험 3: TLB 미스 측정 (perf)

```bash
# TLB 관련 하드웨어 카운터 측정
$ perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses \
  -p $PID sleep 10

# 출력 예시:
# 1,234,567,890  dTLB-loads          (데이터 TLB 조회 총 횟수)
#     2,345,678  dTLB-load-misses    (TLB 미스 횟수)
# TLB 미스율 = 2,345,678 / 1,234,567,890 ≈ 0.19%

# HugePages 활성화 전후 비교
$ echo always > /sys/kernel/mm/transparent_hugepage/enabled
$ perf stat -e dTLB-load-misses -p $PID sleep 10
# vs
$ echo never > /sys/kernel/mm/transparent_hugepage/enabled
$ perf stat -e dTLB-load-misses -p $PID sleep 10
```

---

## 📊 성능/비용 비교

| 항목 | 4KB 페이지 | 2MB HugePages |
|------|-----------|--------------|
| TLB 커버리지 (512 엔트리) | 2MB | 1GB |
| TLB 미스율 (대용량 Heap) | 높음 | 낮음 |
| CoW 복사 단위 (BGSAVE 등) | 4KB | 2MB |
| 메모리 단편화 | 적음 | 클 수 있음 |
| 적합한 워크로드 | Redis (CoW 많음) | JVM (Heap 대용량) |

**페이지 테이블 자체의 메모리 오버헤드**:

```
4KB 페이지로 8GB Heap 관리 시:
  필요한 PTE 수: 8GB / 4KB = 2,097,152 개
  PTE 하나 크기: 8 bytes
  페이지 테이블 크기: 16MB (Heap의 0.2%)

2MB HugePages로 8GB Heap 관리 시:
  필요한 PTE 수: 8GB / 2MB = 4,096 개
  페이지 테이블 크기: 32KB (512배 감소!)
```

---

## ⚖️ 트레이드오프

**가상 메모리의 오버헤드**

가상 메모리는 격리와 유연성을 제공하지만 변환 비용이 있다. 모든 메모리 접근마다 TLB 조회가 필요하고, TLB 미스 시 Page Table Walk(메모리 4번 접근)가 발생한다. 메모리 집약적 워크로드에서 TLB 미스율이 성능의 핵심 변수가 된다.

**THP(Transparent HugePages)의 딜레마**

THP는 커널이 자동으로 4KB 페이지를 2MB HugePages로 병합한다. JVM처럼 대용량 Heap을 쓰는 애플리케이션은 TLB 효율이 올라간다. 반면 Redis처럼 CoW를 활용하는 서비스는 BGSAVE 중 2MB 단위 복사로 메모리가 폭증한다. 단일 설정으로 모든 워크로드를 최적화할 수 없어 서비스별로 다르게 설정해야 한다.

---

## 📌 핵심 정리

```
가상 메모리:
  목적: 격리, 단편화 방지, Demand Paging
  구현: 4단계 Page Table (PGD→PUD→PMD→PTE→물리주소)

TLB:
  CPU 내부 주소 변환 캐시 (~1 사이클)
  미스 시: Page Table Walk (수십~수백 사이클)
  프로세스 스위칭 시 플러시 (성능 저하 원인)

HugePages (2MB):
  TLB 커버리지 512배 증가
  JVM: 유리 (대용량 Heap, CoW 적음)
  Redis: 불리 (BGSAVE CoW 2MB 단위)
  → Redis: THP=never 설정 필수

핵심 진단:
  VSZ ≠ 실제 메모리 / RSS = 실제 물리 메모리
  /proc/<pid>/maps       → VMA 목록
  /proc/<pid>/smaps      → 영역별 RSS, AnonHugePages
  /proc/<pid>/status     → VmRSS, VmSize 요약
  perf stat dTLB-load-misses → TLB 미스율
```

---

## 🤔 생각해볼 문제

**Q1.** 두 프로세스가 같은 공유 라이브러리(`libc.so`)를 사용한다. 각 프로세스의 Page Table에는 이 라이브러리에 대한 PTE가 있다. 물리 메모리는 몇 벌인가? `smaps`의 어떤 값으로 확인할 수 있는가?

<details>
<summary>해설 보기</summary>

물리 메모리는 **1벌**이다. 두 프로세스의 PTE가 같은 물리 페이지 프레임을 가리킨다. `smaps`의 `Shared_Clean` 항목으로 확인할 수 있다. 공유 라이브러리 코드 페이지는 읽기 전용이므로 CoW 없이 영구 공유된다. `Pss`(Proportional Set Size)는 공유 페이지를 프로세스 수로 나눠 각 프로세스의 실제 점유 비용을 나타낸다.

</details>

**Q2.** Java 프로세스가 `-Xmx4g`로 시작했다. 시작 직후 `free -h`에서 used 메모리는 얼마나 증가하는가?

<details>
<summary>해설 보기</summary>

Demand Paging 때문에 **시작 직후에는 거의 증가하지 않는다.** JVM이 4GB를 가상 메모리로 예약하지만, 실제 물리 메모리는 실제로 접근한 페이지만 할당된다. JVM 시작 시 코드, 메타스페이스, 초기 Heap 일부만 접근되므로 수백 MB 수준이다. 시간이 지나 애플리케이션이 Heap을 실제로 사용하면 RSS가 증가한다. `watch cat /proc/$PID/status | grep VmRSS`로 실시간 확인할 수 있다.

</details>

**Q3.** Redis에서 `CONFIG SET hz 100`으로 설정하면 TLB와 어떤 관계가 있는가?

<details>
<summary>해설 보기</summary>

직접적인 관계는 없지만 간접적 영향이 있다. `hz`를 높이면 Redis의 주기적 만료 검사(Active Expiry)가 더 자주 실행되어 더 많은 키가 삭제된다. 삭제된 키의 메모리가 jemalloc에 반환되고 `munmap()`이 호출될 수 있다. `munmap()`은 TLB 플러시를 유발한다. TLB 플러시 후 첫 메모리 접근에서 TLB 미스가 발생해 미세한 레이턴시 증가가 생길 수 있다. 대부분의 경우 무시할 수준이지만 지연에 매우 민감한 환경에서는 고려 사항이 된다.

</details>

---

**[홈으로 🏠](../README.md)** | **[다음: Page Fault ➡️](./02-page-fault.md)**
