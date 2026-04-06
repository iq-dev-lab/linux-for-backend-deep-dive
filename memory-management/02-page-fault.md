# 02. Page Fault — 물리 메모리가 할당되는 시점

## 🎯 핵심 질문

- `malloc()`을 호출하면 물리 메모리가 즉시 할당되는가?
- Minor Page Fault와 Major Page Fault는 어떻게 다른가?
- Page Fault가 성능에 미치는 영향은 얼마나 큰가?
- JVM GC와 Page Fault는 어떤 관계인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Java 서비스를 재시작하면 처음 몇 분간 응답이 느리다.** JVM Warm-up 문제로 JIT 컴파일을 탓하는 경우가 많지만, Page Fault도 주요 원인이다. JVM이 Heap을 처음 사용할 때 수백만 번의 Page Fault가 발생하고, 각각 커널 개입이 필요하다. `-XX:+AlwaysPreTouch`로 JVM 시작 시 Heap 전체에 미리 접근(Page Fault 선발생)하면 서비스 중 지연을 줄일 수 있다.

**`mlock()`을 사용하지 않은 Redis가 스왑 영역으로 밀려난다.** 물리 메모리가 부족하면 커널이 사용하지 않는 페이지를 디스크 스왑으로 내보낸다. Redis가 해당 페이지에 접근하면 Major Page Fault(디스크 I/O)가 발생해 수십 ms 레이턴시가 생긴다. Redis가 `vm-enabled no`를 기본으로 하고 `mlock()`으로 메모리를 고정하는 이유다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: malloc() 후 메모리가 즉시 확보됐다고 가정
# C 코드에서:
char *buf = malloc(1024 * 1024 * 1024); // 1GB
// "이제 1GB 확보됨"
// → 실제로는 가상 주소만 예약, 물리 메모리 할당 안 됨
// → 첫 접근 시 Page Fault 발생, 성능 예측 불가

# 실수 2: Java 서비스 재시작 후 성능 저하를 JIT 탓으로만 돌림
# "몇 분 지나면 JIT가 워밍업되니까 기다리면 돼"
# → Page Fault 워밍업도 동시에 발생
# → AlwaysPreTouch로 시작 시간에 비용 선지불 가능

# 실수 3: 스왑 공간 없이 운영
# "스왑은 느리니까 없애면 돼"
# → OOM Killer가 바로 발동 (스왑으로 버틸 여유 없음)
# → 스왑은 완충지대. 있어야 하되 사용 중이면 경고 신호
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# JVM 시작 시 Heap 미리 커밋 (Page Fault 선발생)
$ java -XX:+AlwaysPreTouch -Xms4g -Xmx4g -jar app.jar
# 시작이 느려지지만, 서비스 중 Page Fault 제거

# Redis 메모리 고정 (스왑 방지)
# redis.conf:
# maxmemory 6gb
# 그리고 OS 레벨에서:
$ cat /proc/$(pgrep redis-server)/status | grep VmSwap
VmSwap: 0 kB   ← 스왑 사용 없어야 정상

# 스왑 사용 중인 프로세스 찾기
$ for pid in $(ls /proc | grep -E '^[0-9]+$'); do
    swap=$(awk '/VmSwap/{print $2}' /proc/$pid/status 2>/dev/null)
    [ "${swap:-0}" -gt 0 ] && echo "PID=$pid Swap=${swap}kB $(cat /proc/$pid/comm 2>/dev/null)"
  done | sort -t= -k3 -rn | head -10
```

---

## 🔬 내부 동작 원리

### Demand Paging — 필요할 때만 할당

```
malloc(1GB) 호출:

1. libc의 malloc() → mmap() 시스템 콜
2. 커널: VMA(Virtual Memory Area) 생성
   → 가상 주소 공간에 1GB 영역 예약
   → 물리 메모리는 아직 할당하지 않음
   → Page Table 항목: "존재하지 않음" 표시

3. 애플리케이션으로 가상 주소 반환

첫 번째 buf[0] = 1; 실행:
4. CPU가 가상 주소 번역 시도
5. Page Table에 "존재하지 않음" → Page Fault 예외 발생
6. 커널 Page Fault 핸들러 진입
7. 여유 물리 페이지 할당 (free list에서)
8. Page Table 업데이트 (가상 → 물리 매핑)
9. 프로세스 재개, buf[0] = 1 정상 실행

→ 4KB씩 1GB면 262,144번의 Page Fault
```

### Minor Page Fault vs Major Page Fault

```
Minor Page Fault (빠름, ~수 마이크로초):
  원인:
    ① Demand Paging: 처음 접근하는 익명 페이지
    ② 공유 라이브러리 첫 접근 (이미 Page Cache에 있음)
    ③ CoW 발생 시 페이지 복사
  처리: 디스크 I/O 없음
    → 물리 페이지 할당 + Page Table 업데이트만

Major Page Fault (느림, ~수 밀리초):
  원인:
    ① 스왑으로 밀려난 페이지에 다시 접근
    ② mmap()된 파일이 Page Cache에 없을 때 첫 접근
    ③ 실행 파일 로딩 (디스크에서 코드 읽기)
  처리: 디스크 I/O 필수
    → 디스크에서 페이지 읽기 → 물리 메모리 적재 → Page Table 업데이트

비용 비교:
  Minor: ~1~10 μs (물리 페이지 할당 + TLB 업데이트)
  Major: ~1~10 ms (디스크 접근, 1000배 이상 차이)
```

### Page Fault 처리 흐름 (커널 내부)

```
Page Fault 예외 발생
       │
       ▼
do_page_fault() 진입
       │
       ├── VMA 존재 확인 (find_vma())
       │   없으면 → SIGSEGV (Segmentation Fault)
       │
       ├── 접근 권한 확인
       │   읽기 전용 영역에 쓰기 → SIGSEGV
       │
       ├── Page Table 확인
       │   │
       │   ├── PTE = 0 (한 번도 접근 안 함)
       │   │   → do_anonymous_page() (익명 페이지 할당)
       │   │   → Minor Page Fault
       │   │
       │   ├── PTE 있음 + swap bit 설정 (스왑으로 밀려남)
       │   │   → do_swap_page()
       │   │   → 디스크에서 읽기 → Major Page Fault
       │   │
       │   └── PTE 있음 + 쓰기 보호 (CoW)
       │       → do_wp_page()
       │       → 페이지 복사 → Minor Page Fault
       │
       └── 프로세스 재개
```

### 스왑과 Page Fault

```
물리 메모리 부족 시 커널 동작:

1. kswapd (커널 스왑 데몬) 활성화
2. LRU(Least Recently Used) 기반으로 피해자 페이지 선택
3. 더티 페이지 → 디스크 스왑 영역에 기록
4. Page Table에서 해당 항목에 swap bit 설정
5. 물리 페이지 해제 → 다른 프로세스에 할당

프로세스가 스왑된 페이지 접근 시:
→ Major Page Fault
→ 디스크 스왑 영역에서 읽기 (~5~20ms)
→ 물리 메모리에 다시 적재
→ Page Table 업데이트

Redis에서 이 상황이 발생하면:
  명령어 처리 시간이 수십 ms로 폭증
  → 클라이언트 타임아웃
  → "연결 불안정" 오류
```

---

## 💻 실전 실험

### 실험 1: Demand Paging vs AlwaysPreTouch 비교

```bash
# AlwaysPreTouch 없이 (기본)
$ time java -Xms4g -Xmx4g -jar app.jar &
$ sleep 2
$ /usr/bin/time -v cat /proc/$(pgrep java)/status 2>&1 | grep VmRSS
# 초기 RSS: 수백 MB (전체 4GB 아님)

# AlwaysPreTouch 적용
$ time java -XX:+AlwaysPreTouch -Xms4g -Xmx4g -jar app.jar &
$ sleep 2
$ cat /proc/$(pgrep java)/status | grep VmRSS
# 초기 RSS: ~4GB (시작 시간은 더 걸리지만 이후 Page Fault 없음)
```

### 실험 2: Page Fault 횟수 실시간 모니터링

```bash
# pidstat으로 초당 Page Fault 횟수 확인
$ pidstat -r 1 -p $PID
#          minflt/s  majflt/s
# 시작 직후:  50000.0     0.0   ← Minor Page Fault 폭발
# 안정화 후:    100.0     0.0   ← 정상 수준

# perf stat으로 정밀 측정
$ perf stat -e page-faults,major-faults -p $PID sleep 10
#    2,345,678  page-faults     (Minor)
#            3  major-faults    (Major, 3번은 정상)
```

### 실험 3: 스왑 사용 중인 프로세스 확인 및 영향 측정

```bash
# 스왑 현황 확인
$ vmstat 1 5
# si(swap in): 초당 스왑에서 읽어온 KB
# so(swap out): 초당 스왑으로 내보낸 KB
# si > 0이면 Major Page Fault 발생 중

# 프로세스별 스왑 사용량
$ cat /proc/$PID/status | grep VmSwap
VmSwap: 102400 kB  ← 100MB 스왑 사용 중 → 위험!

# 스왑 사용 없애기 (해당 프로세스 메모리 강제 복귀)
$ cat /proc/$PID/status | grep VmSwap
# swapoff -a && swapon -a  → 전체 스왑 플러시 (위험, 운영 주의)

# mlock으로 특정 프로세스 스왑 방지 (Redis 설정)
# redis.conf: activerehashing yes 설정 후
# mlockall() 시스템 콜로 전체 메모리 고정
$ cat /proc/$(pgrep redis)/status | grep VmSwap
VmSwap: 0 kB  ← mlock으로 스왑 방지
```

---

## 📊 성능/비용 비교

| Page Fault 유형 | 발생 원인 | 처리 시간 | 디스크 I/O |
|----------------|---------|---------|-----------|
| Minor (익명) | 첫 접근, Demand Paging | ~1~5 μs | 없음 |
| Minor (CoW) | fork() 후 쓰기 | ~5~20 μs | 없음 |
| Minor (공유 라이브러리) | Page Cache 히트 | ~1~5 μs | 없음 |
| Major (스왑) | 스왑된 페이지 접근 | ~5~20 ms | SSD/HDD |
| Major (mmap 파일) | Page Cache 미스 | ~1~10 ms | SSD |

**AlwaysPreTouch 트레이드오프**:

```
기본 설정 (Demand Paging):
  JVM 시작: 빠름 (수 초)
  서비스 첫 트래픽: 느림 (수십만 Page Fault)
  메모리: 실제 사용량만큼 증가 (효율적)

AlwaysPreTouch:
  JVM 시작: 느림 (Heap 크기 × Page Fault 시간)
  서비스 첫 트래픽: 정상 (Page Fault 없음)
  메모리: 즉시 Heap 전체 확보 (고정 비용)

Kubernetes에서 주의:
  AlwaysPreTouch + Heap 4GB → 시작 시 4GB 메모리 즉시 사용
  → Pod가 메모리 limit 초과 → OOM Kill
  → limit을 실제 최대 사용량 + 여유분으로 설정 필요
```

---

## ⚖️ 트레이드오프

**Demand Paging의 장단점**

장점: 실제 사용하는 메모리만 물리 메모리를 소비해 효율적이다. 수백 개의 프로세스가 각자 큰 가상 주소 공간을 가질 수 있다.

단점: 첫 접근 시 Page Fault 비용이 발생한다. 실시간 서비스에서 첫 요청이 느려지는 Warm-up 문제가 생긴다. 이를 해결하기 위해 AlwaysPreTouch나 Preloading을 사용하면 시작 시 비용이 증가한다.

**스왑 공간의 딜레마**

스왑 없음: OOM 상황에서 OOM Killer가 바로 발동해 프로세스를 종료한다. 스왑이 완충지대 역할을 못 해 갑작스러운 서비스 중단이 발생한다.

스왑 있음: OOM 상황에서 버티는 시간이 늘어난다. 그러나 스왑 사용이 시작되면 Major Page Fault로 성능이 급격히 저하된다. 스왑 사용 자체가 메모리 부족의 신호이므로 알림을 설정해야 한다.

---

## 📌 핵심 정리

```
Demand Paging:
  malloc() = 가상 주소 예약만 (물리 메모리 할당 안 됨)
  첫 접근 시 → Page Fault → 커널이 물리 페이지 할당

Minor Page Fault:
  디스크 I/O 없음, ~1~10 μs
  원인: 첫 접근(익명 페이지), CoW, 공유 라이브러리

Major Page Fault:
  디스크 I/O 필수, ~1~20 ms
  원인: 스왑된 페이지 접근, mmap 파일 캐시 미스

JVM 연계:
  AlwaysPreTouch: 시작 시 Page Fault 선발생 → 서비스 중 안정적
  G1GC humongous allocation: 대형 객체 할당 시 Major Fault 가능

Redis 연계:
  스왑 사용 → Major Page Fault → 레이턴시 수십 ms
  mlock() / vm-enabled no로 스왑 방지

핵심 진단 명령어:
  /usr/bin/time -v    → Major/Minor Page Fault 횟수
  pidstat -r 1        → 초당 Page Fault 실시간
  vmstat si/so        → 스왑 I/O 발생 여부
  /proc/<pid>/status VmSwap → 프로세스 스왑 사용량
```

---

## 🤔 생각해볼 문제

**Q1.** `fork()` 호출 후 부모와 자식 모두 Heap 데이터를 읽기만 한다. 이때 Minor Page Fault가 발생하는가?

<details>
<summary>해설 보기</summary>

**발생하지 않는다.** `fork()` 직후 부모와 자식은 같은 물리 페이지를 공유하며, 이미 Page Table에 매핑이 되어 있다. 읽기만 한다면 CoW도 발생하지 않으므로 추가 Page Fault가 없다. 부모나 자식이 페이지에 쓰기를 시도할 때 CoW로 인한 Minor Page Fault가 발생한다.

</details>

**Q2.** Java GC가 실행될 때 Major Page Fault가 발생할 수 있는가?

<details>
<summary>해설 보기</summary>

**그렇다.** GC가 오래된 객체(Old Generation)를 스캔할 때, 해당 객체가 스왑으로 밀려나 있다면 Major Page Fault가 발생한다. GC 도중 Major Page Fault가 빈번하면 GC 시간이 수십 배로 늘어나 STW(Stop-The-World) 시간이 폭발한다. `jstat -gcutil`에서 GC 시간이 비정상적으로 길다면 `vmstat`의 `si` 값을 함께 확인해야 한다.

</details>

**Q3.** `mmap()`으로 10GB 파일을 매핑했다. 시스템에 물리 메모리가 8GB밖에 없다. 이 파일 전체를 읽으면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

처음 8GB까지는 Minor/Major Page Fault로 정상 읽기가 가능하다. 이후 커널의 Page Cache Eviction이 발생해 오래된 페이지를 해제하고 새 페이지를 적재한다. 파일 전체를 순차적으로 읽는다면 커널의 Read-Ahead(미리 읽기)가 동작해 Major Page Fault를 줄인다. 메모리보다 큰 파일을 `mmap()`으로 처리하는 것은 커널이 자동으로 페이지 교체를 관리해주므로 실용적이다. 단, 랜덤 접근 패턴에서는 Page Cache 히트율이 낮아 성능이 저하된다.

</details>

---

**[⬅️ 이전: 가상 메모리](./01-virtual-memory.md)** | **[홈으로 🏠](../README.md)** | **[다음: Page Cache ➡️](./03-page-cache.md)**
