# 02. 메모리 분석 — cache/buffer의 실체와 누수 진단

## 🎯 핵심 질문

- `free -h`의 `available`이 `free + buff/cache`와 다른 이유는?
- `vmstat`의 `si`/`so`(swap in/out)가 높을 때 무엇을 의미하는가?
- `/proc/meminfo`의 어떤 항목들이 메모리 누수 진단에 핵심인가?
- RSS가 계속 증가하는 프로세스의 누수 위치를 어떻게 좁히는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**`free -h`에서 `used`가 14GB인데 `available`이 13GB다.** `buff/cache`가 12GB를 사용하고 있는 것이다. 새 애플리케이션이 메모리를 요청하면 커널이 이 캐시를 해제해 제공한다. 메모리 부족이 아니다. `available` 값이 실제로 사용 가능한 메모리다.

**Java 서비스의 RSS가 매일 100MB씩 증가한다.** JVM Heap은 GC로 정리되는데 RSS가 증가한다면 Native Memory 누수를 의심해야 한다. Metaspace, Direct Buffer, JNI 네이티브 라이브러리 등이 대상이다. `jcmd VM.native_memory`와 `/proc/<pid>/smaps` 분석이 출발점이다.

**`vmstat`에서 `si`(swap in)가 계속 발생한다.** 물리 메모리가 부족해 디스크 스왑에서 페이지를 읽어오고 있다. 이 상태에서는 대부분의 메모리 접근에 수 ms 지연이 추가된다. 스왑이 발생하는 프로세스를 찾아 메모리를 확보해야 한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: used 보고 메모리 부족 판단
$ free -h
              total  used   free  buff/cache  available
Mem:           15Gi  14Gi  512Mi       12Gi       13Gi
# "14GB 사용 중이야! 메모리 부족!" → 오해
# available 13GB → 실제로 사용 가능한 메모리는 충분
# buff/cache 12GB는 Page Cache (필요 시 즉시 반환)

# 실수 2: 캐시 비우기로 해결 시도
$ echo 3 > /proc/sys/vm/drop_caches
# → 일시적으로 used 감소, buff/cache 감소
# → 이후 파일 접근 시 다시 캐시 채움
# → 실제 메모리 부족 문제 해결 안 됨, 오히려 성능 저하

# 실수 3: RSS만 보고 누수 판단
$ ps aux | grep java
java  1234  2.0  30.0  8388608  491520  ← RSS 480MB
# "30% 사용 중" → JVM Heap 점유는 정상일 수 있음
# 중요한 것: 시간에 따른 RSS 변화 추이
# 지속적 증가 = 누수 의심
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 메모리 상태 단계별 진단

# Step 1: 전체 현황
$ free -h  # available이 핵심 지표

# Step 2: 스왑 사용 확인
$ vmstat 1 5
# si/so 컬럼: 0이어야 정상, 값 있으면 스왑 발생 중

# Step 3: 프로세스별 메모리 사용
$ ps aux --sort=-%mem | head -10
# 또는 top에서 M 키로 메모리 정렬

# Step 4: RSS 누수 의심 시 추이 관찰
$ while true; do
    echo "$(date) $(ps -o rss= -p $PID)KB"
    sleep 60
  done

# Step 5: 누수 위치 좁히기
$ cat /proc/$PID/smaps_rollup
# 또는
$ jcmd $PID VM.native_memory summary  # Java
```

---

## 🔬 내부 동작 원리

### /proc/meminfo 핵심 항목 완전 해석

```bash
$ cat /proc/meminfo

MemTotal:       16384000 kB  ← 물리 메모리 전체
MemFree:          512000 kB  ← 아무것도 없는 완전한 빈 메모리
MemAvailable:   13312000 kB  ← ★ 실제 사용 가능 (앱이 쓸 수 있음)
                              (free + 회수 가능한 캐시 - 최소 예약분)
Buffers:          102400 kB  ← 블록 장치 메타데이터 캐시 (작음)
Cached:         12288000 kB  ← 파일 Page Cache (회수 가능!)
SwapCached:           0 kB  ← 스왑에서 읽었지만 다시 RAM에 있는 것

Active:          8192000 kB  ← 최근 사용된 페이지 (회수 어려움)
Inactive:        5120000 kB  ← 덜 사용된 페이지 (회수 후보)

Dirty:            204800 kB  ← 수정됐지만 아직 디스크에 안 쓴 페이지
Writeback:          4096 kB  ← 현재 디스크에 쓰는 중인 페이지
AnonPages:       2048000 kB  ← 파일 아닌 익명 페이지 (Heap, 스택 등)

Mapped:           819200 kB  ← mmap으로 매핑된 페이지
Shmem:            102400 kB  ← 공유 메모리 (tmpfs 포함)

Slab:             512000 kB  ← 커널 객체 캐시 (inode, dentry 등)
SReclaimable:     409600 kB  ← 회수 가능한 Slab (캐시)
SUnreclaim:       102400 kB  ← 회수 불가 Slab (사용 중 커널 객체)

VmallocTotal:  34359738367 kB  ← vmalloc 가상 공간 (64비트에서 매우 큼)
VmallocUsed:       65536 kB   ← 실제 vmalloc 사용량

HugePages_Total:   0           ← HugePages 전체 수
HugePages_Free:    0           ← 여유 HugePages

메모리 가용성 계산:
  MemAvailable = MemFree + (Cached - 최소유지분) + SReclaimable
  → buff/cache가 있어도 available이 크면 메모리 여유 충분
```

### vmstat 스왑 지표 해석

```bash
$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  1 102400 512000 102400 4096000   50  100   200   300  800 2000  8 15 70  7  0
         ↑↑↑↑↑                     ↑↑↑  ↑↑↑
         스왑 사용 중               in   out

si (swap in):  디스크 스왑 → RAM으로 읽음 (페이지 fault)
so (swap out): RAM → 디스크 스왑으로 내보냄

해석:
  si=0, so=0: 스왑 없음 (정상)
  si>0, so=0: 이전에 스왑된 페이지 접근 중 (일시적일 수 있음)
  si>0, so>0: 지속적 스왑 활동 (메모리 부족 심각)
  so>0, si=0: RAM 부족으로 내보내는 중

si/so > 0일 때 조치:
  1. 어느 프로세스가 스왑에 있는지 확인
     $ for pid in $(ls /proc | grep -E '^[0-9]+$'); do
         swap=$(awk '/VmSwap/{print $2}' /proc/$pid/status 2>/dev/null)
         [ "${swap:-0}" -gt 0 ] && echo "PID=$pid Swap=${swap}kB"
       done | sort -t= -k3 -rn | head -5
  2. 해당 프로세스 메모리 줄이기 또는 서버 메모리 증설
  3. 스왑 사용 원천 차단: mlock() 또는 mlockall()
```

### 메모리 누수 유형과 진단

```
메모리 누수 유형:

유형 1: Heap 누수 (Java GC leak)
  증상: GC 후에도 메모리 회수 안 됨, GC 빈도 증가
  확인: jstat -gcutil <pid> 1
        GC 후 Old Gen 사용률 계속 증가
  도구: Heap Dump → jmap -dump:format=b,file=heap.hprof <pid>
        분석: Eclipse MAT, VisualVM

유형 2: Native Memory 누수 (Off-heap)
  증상: RSS 증가하지만 JVM Heap은 정상
  확인: jcmd <pid> VM.native_memory summary
        Internal 또는 Other 항목 지속 증가
  도구: NMT baseline + diff
        $ jcmd <pid> VM.native_memory baseline
        ... (시간 경과) ...
        $ jcmd <pid> VM.native_memory detail.diff

유형 3: C/C++ 네이티브 코드 누수
  증상: RSS 증가, JVM 관련 메모리는 정상
  확인: valgrind --leak-check=full ./app
  또는: /proc/<pid>/smaps_rollup 추이 관찰

유형 4: 파일 디스크립터 누수 (간접적 메모리 영향)
  증상: "Too many open files", fd 수 지속 증가
  확인: ls /proc/<pid>/fd | wc -l
        lsof -p <pid> | wc -l
```

### /proc/pid/smaps 분석

```bash
$ cat /proc/$PID/smaps | head -30
7f8a00000000-7f8b00000000 rw-p 00000000 00:00 0  [heap]
Size:          1048576 kB  ← 예약된 가상 크기
Rss:            204800 kB  ← 실제 물리 메모리 (이것이 실제 점유)
Pss:            204800 kB  ← 공유 고려한 비례 크기
Shared_Clean:        0 kB
Shared_Dirty:        0 kB
Private_Clean:       0 kB
Private_Dirty:  204800 kB  ← 이 프로세스만 쓰는 수정된 페이지
Referenced:     204800 kB  ← 최근 접근된 페이지
Anonymous:      204800 kB  ← 파일 아닌 익명 메모리 (Heap)
AnonHugePages:       0 kB  ← THP로 할당된 크기
KSM:                 0 kB
LazyFree:            0 kB
Swap:                0 kB  ← 스왑으로 이동된 크기
SwapPss:             0 kB

# 전체 요약
$ cat /proc/$PID/smaps_rollup
5f3c00000000-7ffee2000000 ---p 00000000 00:00 0
Rss:            491520 kB   ← 전체 RSS (이것이 실제 물리 메모리)
Pss:            487424 kB
Private_Dirty:  401408 kB   ← 이 프로세스만 사용하는 더티 페이지
Swap:                0 kB

# 영역별 분류 (누수 위치 특정)
$ awk '/^[0-9a-f]/{name=$NF} /^Rss:/{print $2, name}' \
  /proc/$PID/smaps | sort -rn | head -10
```

---

## 💻 실전 실험

### 실험 1: 메모리 부족 vs 캐시 구분

```bash
# 대용량 파일 읽기로 Page Cache 채우기
$ dd if=/dev/urandom of=/tmp/bigfile bs=1M count=2048
$ cat /tmp/bigfile > /dev/null  # 2GB Page Cache 채움

$ free -h
# buff/cache: 2GB 증가 (Page Cache)
# available: 여전히 큼 (회수 가능하므로)

$ cat /proc/meminfo | grep -E "MemAvailable|Cached|Dirty"
# Cached: 2GB 정도 증가
# MemAvailable: 거의 변화 없음 (캐시는 회수 가능)

# 실제 메모리 부족 시뮬레이션
$ stress-ng --vm 1 --vm-bytes 90% --timeout 10s &
$ free -h  # available이 크게 감소
$ vmstat 1 5  # si/so 확인 (스왑 발생?)
```

### 실험 2: RSS 증가 추적

```bash
# 메모리 누수 시뮬레이션 프로그램
$ cat > /tmp/leak.c << 'EOF'
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    while(1) {
        void *p = malloc(1024 * 1024);  // 1MB 할당
        // free(p) 없음 → 누수!
        printf("Allocated 1MB, total RSS growing...\n");
        sleep(1);
    }
}
EOF
$ gcc /tmp/leak.c -o /tmp/leak && /tmp/leak &
PID=$!

# RSS 추이 관찰
$ while kill -0 $PID 2>/dev/null; do
    rss=$(awk '/VmRSS/{print $2}' /proc/$PID/status)
    echo "$(date +%H:%M:%S) RSS=${rss}kB"
    sleep 2
  done

# smaps에서 누수 위치 확인
$ grep -A 5 "\[heap\]" /proc/$PID/smaps
# Private_Dirty가 계속 증가하는 것 확인

kill $PID
```

### 실험 3: Java Native Memory 누수 추적

```bash
# NMT 활성화
$ java -XX:NativeMemoryTracking=detail \
       -XX:+UseContainerSupport \
       -jar app.jar &
PID=$(pgrep java)

# 기준선 설정
$ jcmd $PID VM.native_memory baseline

# 부하 실행 (5분)
# ...

# 변화 확인
$ jcmd $PID VM.native_memory detail.diff
# 출력에서 증가한 영역 확인:
# +5MB: Internal (Metaspace 관련)
# +10MB: Other (JNI 네이티브)
# → 어느 영역이 누수인지 좁힘
```

---

## 📊 성능/비용 비교

**메모리 상태 판단 기준**:

```
지표              정상              주의              심각
MemAvailable      > 20%           10~20%             < 10%
Dirty(+Writeback) < 5% of total  5~10%              > 10%
vmstat si/so      = 0             가끔 발생          지속 발생
VmSwap (프로세스) = 0             수 MB              수십 MB+
```

**메모리 누수 진단 도구 비교**:

```
도구                    대상        오버헤드  정밀도
/proc/pid/smaps_rollup  모든 프로세스  없음     낮음 (전체 RSS)
jcmd VM.native_memory   Java        낮음     중간 (영역별)
valgrind               C/C++       20~50배   높음 (할당 추적)
jemalloc heap profiler  C/C++       낮음     중간 (통계 기반)
Eclipse MAT            Java Heap   분석 시만  높음 (객체별)
```

---

## ⚖️ 트레이드오프

**Dirty 비율과 데이터 안전성**

Dirty 페이지(수정됐지만 아직 디스크에 안 쓴 페이지)가 많을수록 쓰기 처리량은 좋아지지만, 장애 시 데이터 손실 위험이 커진다. `vm.dirty_ratio`(기본 20%)를 낮추면 데이터가 더 자주 플러시되어 안전하지만 쓰기 I/O가 증가한다. MySQL/PostgreSQL은 WAL(Write-Ahead Log)로 이 문제를 직접 관리하므로 OS 설정보다 DB 설정이 우선이다.

**Slab 캐시의 양면성**

`/proc/meminfo`의 `Slab`은 커널이 inode, dentry, 네트워크 소켓 구조체 등을 캐싱한 것이다. `SReclaimable`은 회수 가능해 `MemAvailable`에 포함되지만, `SUnreclaim`은 회수 불가다. 대규모 파일 시스템 작업(수백만 개 파일) 후 `Slab`이 수 GB를 차지하는 경우가 있다. `echo 2 > /proc/sys/vm/drop_caches`로 reclaimable slab을 해제할 수 있지만 성능 저하를 유발한다.

---

## 📌 핵심 정리

```
free -h 해석:
  available = 실제 사용 가능 메모리 (★ 이 값이 핵심)
  buff/cache = Page Cache (회수 가능, 부족 아님)
  free ≪ available은 정상 (캐시가 많이 쌓인 것)

vmstat 스왑 지표:
  si(swap in) = 스왑에서 RAM으로 → 이전 스왑 페이지 접근
  so(swap out) = RAM에서 스왑으로 → 메모리 부족 신호
  → 0이어야 정상, 지속 발생 시 메모리 증설 또는 프로세스 메모리 감소

/proc/meminfo 주요 항목:
  Dirty       → 아직 안 쓴 더티 페이지 (대용량이면 flush 지연)
  AnonPages   → Heap, 스택 등 익명 메모리
  Slab        → 커널 객체 캐시

메모리 누수 진단 흐름:
  RSS 지속 증가 확인 (ps aux, /proc/<pid>/status VmRSS)
  smaps_rollup으로 어느 영역이 증가하는지
  Java: jcmd VM.native_memory diff
  C/C++: valgrind 또는 /proc/<pid>/maps 분석

핵심 명령어:
  free -h                          → 전체 메모리 현황
  vmstat 1                         → 스왑 in/out 확인
  cat /proc/meminfo                 → 상세 분류
  cat /proc/<pid>/smaps_rollup      → 프로세스 메모리 요약
  jcmd <pid> VM.native_memory diff  → Java 누수 추적
```

---

## 🤔 생각해볼 문제

**Q1.** Kafka 브로커의 JVM Heap 사용률은 40%인데 시스템 메모리는 95%다. 무엇을 의심하고 어떻게 확인하는가?

<details>
<summary>해설 보기</summary>

Kafka는 Page Cache를 적극 활용한다. JVM Heap 외에 Kafka가 읽고 쓰는 로그 파일이 Page Cache를 채우고 있을 가능성이 높다. 이것은 정상적인 Kafka 동작이다. `free -h`에서 `available`이 충분하면 문제없다. 확인 방법: `free -h`로 `available` 확인, `/proc/meminfo`에서 `Cached` 항목 확인(Kafka 로그 Page Cache). 만약 `available`이 낮고 스왑이 발생한다면 문제다. 이 경우 JVM Heap을 줄이거나(`-Xmx`), 로그 보존 기간을 단축하거나 파티션 수를 줄이는 것을 검토한다.

</details>

**Q2.** `vmstat`에서 `so`가 없는데 `si`가 지속 발생한다. 어떤 상황인가?

<details>
<summary>해설 보기</summary>

이전에 스왑으로 내보낸 페이지를 다시 읽어오는 상황이다. 현재는 메모리가 여유 있어서 스왑 아웃(`so`)은 발생하지 않지만, 이전에 스왑에 내보낸 페이지가 있고 해당 페이지에 접근할 때 스왑 인(`si`)이 발생한다. 오래전에 스왑된 프로세스가 다시 활성화되는 패턴이다. `si > 0`이면 Major Page Fault가 발생해 디스크 I/O가 생기므로 해당 프로세스의 응답이 느려질 수 있다. `for pid in $(ls /proc | grep '^[0-9]'); do ...; done`으로 스왑 사용 중인 프로세스를 찾고 메모리를 확보한다.

</details>

**Q3.** `/proc/meminfo`의 `SUnreclaim`이 2GB로 크다. 이것을 줄이는 방법이 있는가?

<details>
<summary>해설 보기</summary>

`SUnreclaim`은 커널이 현재 사용 중인 슬랩 캐시다(소켓 구조체, 파일 핸들 등). 이것은 정의상 회수 불가능하다. 줄이려면 해당 커널 객체를 사용하는 프로세스가 종료하거나 소켓을 닫아야 한다. 네트워크 집약적인 서버에서 `SUnreclaim`이 큰 것은 대량의 소켓 구조체를 유지하기 때문이다. 연결 수를 줄이거나(Keep-Alive 활용, 연결 풀 제한), 메모리 증설이 근본 해결책이다. `echo 2 > /proc/sys/vm/drop_caches`로는 `SUnreclaim`은 줄어들지 않는다(SReclaimable만 해제).

</details>

---

**[⬅️ 이전: CPU 분석](./01-cpu-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: I/O 분석 ➡️](./03-io-analysis.md)**
