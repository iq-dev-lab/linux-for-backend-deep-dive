# 05. 메모리 할당기 — ptmalloc vs jemalloc vs tcmalloc

## 🎯 핵심 질문

- `malloc()`을 호출하면 커널에 직접 요청하는가, 아니면 중간 레이어가 있는가?
- 메모리 단편화란 무엇이고, 왜 장기 실행 서버에서 문제가 되는가?
- Redis가 jemalloc을 선택한 이유는?
- JVM이 자체 Heap을 관리하는 이유는 OS 할당기를 믿지 않기 때문인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Redis가 `MEMORY USAGE key`로 보고하는 사용량과 `INFO memory`의 `used_memory_rss`가 크게 다르다.** `used_memory`는 Redis가 실제 저장한 데이터 크기고, `used_memory_rss`는 OS가 Redis 프로세스에 할당한 물리 메모리다. 그 차이가 메모리 단편화다. `mem_fragmentation_ratio = used_memory_rss / used_memory`가 1.5를 넘으면 jemalloc의 단편화 문제를 의심해야 한다.

**Java 서비스가 Heap 사용량은 낮은데 OS 레벨 메모리 사용량이 계속 증가한다.** Off-heap 메모리(Direct Buffer, Metaspace, JIT 코드 캐시)나 Native 라이브러리가 ptmalloc을 직접 사용하면서 단편화가 축적되는 경우다. `jcmd VM.native_memory`로 확인하고, 필요 시 tcmalloc으로 교체할 수 있다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: used_memory만 보고 Redis 메모리 판단
$ redis-cli info memory | grep used_memory
used_memory: 4294967296        # 4GB
used_memory_rss: 8589934592    # 8GB (RSS)
mem_fragmentation_ratio: 2.00  # ← 단편화 심각!

# "데이터는 4GB인데 왜 서버 메모리를 8GB 쓰지?"
# → 단편화 이해 없이 메모리 증설만 반복

# 실수 2: Redis 재시작으로만 단편화 해결 시도
$ redis-cli FLUSHALL  # 데이터 삭제
# → 메모리가 OS에 반환되지 않을 수 있음 (ptmalloc 동작)
# → jemalloc은 페이지 단위로 반환, ptmalloc은 top chunk만 반환

# 실수 3: Java Native Memory 누수를 Heap 문제로 오인
$ jstat -gcutil $PID
  S0    S1    E     O     M    CCS   YGC  YGCT  FGC  FGCT
0.00  0.00 50.00 45.00 95.00 90.00  100  2.500   2  0.300
# Heap 사용률 정상인데 OS 메모리는 계속 증가
# → jcmd로 Native Memory 확인 필요
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# Redis 단편화 상태 정확히 진단
$ redis-cli info memory | grep -E "used_memory|mem_fragmentation|allocator"
used_memory: 4294967296            # Redis가 사용하는 데이터 크기
used_memory_rss: 5368709120        # OS가 할당한 물리 메모리
mem_fragmentation_ratio: 1.25      # 1.0~1.5: 정상, >1.5: 주의, >2.0: 심각
mem_allocator: jemalloc-5.3.0      # 사용 중인 할당기 확인

# Redis 4.0+ 능동적 단편화 해소
$ redis-cli config set activedefrag yes
$ redis-cli config set active-defrag-ignore-bytes 100mb
$ redis-cli config set active-defrag-enabled yes

# Java Native Memory 전체 현황
$ jcmd $PID VM.native_memory summary
Total: reserved=5120MB, committed=2048MB
- Java Heap    (reserved=2048MB, committed=2048MB)
- Class        (reserved=1056MB, committed=52MB)
- Thread       (reserved=400MB,  committed=400MB)
- Code         (reserved=256MB,  committed=40MB)
- GC           (reserved=200MB,  committed=120MB)
- Internal     (reserved=100MB,  committed=100MB)  ← 여기가 늘면 Native 누수 의심
```

---

## 🔬 내부 동작 원리

### malloc()의 두 단계

애플리케이션의 `malloc()`은 커널에 직접 요청하지 않는다. 할당기가 미리 큰 메모리 블록을 확보해두고, 요청을 내부적으로 분할해서 제공한다.

```
애플리케이션
  malloc(100)
       │
       ▼
  할당기 (ptmalloc / jemalloc / tcmalloc)
       │
       ├── 내부 free list에 적합한 블록 있음?
       │   → 있으면: 즉시 반환 (시스템 콜 없음)
       │
       └── 없으면: OS에 메모리 요청
           ├── brk() : 힙 크기 확장 (작은 할당)
           └── mmap(): 큰 블록 요청 (일반적으로 128KB 이상)
                       → 나중에 munmap()으로 OS에 반환

free()도 즉시 OS에 반환 안 함:
  → 할당기 내부 free list에 추가
  → 다음 malloc() 요청에 재사용
  → 이것이 단편화의 출발점
```

### 메모리 단편화 발생 과정

```
외부 단편화 (External Fragmentation):

  초기 상태:
  ┌────────────────────────────────────────┐
  │           빈 메모리 (연속)                │
  └────────────────────────────────────────┘

  할당 후:
  ┌──────┬──────┬──────┬──────┬──────┬───┐
  │A(8B) │B(16B)│C(8B) │D(32B)│E(8B) │빈 │
  └──────┴──────┴──────┴──────┴──────┴───┘

  B와 D를 해제:
  ┌──────┬──────┬──────┬──────┬──────┬───┐
  │A(8B) │ 빈16  │C(8B) │ 빈32 │E(8B) │빈  │
  └──────┴──────┴──────┴──────┴──────┴───┘
  
  "30B 할당 요청" → 가장 큰 빈 블록(32B)에 할당
  "15B 할당 요청" → 16B 블록에 할당
  "20B 할당 요청" → ? 연속된 20B 빈 블록 없음!
                    → mmap()으로 새 블록 요청
                    → 실제 빈 메모리가 있어도 못 쓰는 상황

내부 단편화 (Internal Fragmentation):
  "9B 요청" → 할당기가 16B 블록으로 반올림해 제공
  → 7B 낭비 (블록 내부 사용 안 하는 공간)
```

### ptmalloc (glibc 기본 할당기)

```
ptmalloc의 Arena 구조:

  Main Arena (메인 스레드)
  └── Heap: brk()로 관리되는 연속 메모리 영역
      └── bin[]로 크기별 free list 관리

  Thread Arena (각 스레드, 최대 8 * CPU 코어 수)
  └── Heap: mmap()으로 별도 메모리 영역
      └── 스레드 로컬 → 락 경합 줄임

ptmalloc의 약점:
  1. Main Arena의 "top chunk" 문제:
     Heap의 끝 블록(top chunk)이 해제돼야 brk()로 OS에 반환 가능
     중간 블록이 살아있으면 OS 반환 안 됨 → "메모리 홀딩"

  2. 크기별 bin 분류가 거칠어 내부 단편화 발생
     실제 요청: 예측 불가한 크기들
     bin 크기: 8, 16, 24, ... 등 고정 간격

  3. 장기 실행 서버에서 단편화 누적
     Redis처럼 다양한 크기의 키-값을 지속적으로 추가/삭제하면
     단편화 비율이 계속 증가
```

### jemalloc (Redis 기본 할당기)

```
jemalloc의 핵심 개선점:

1. Arena + Thread Cache 구조
   각 CPU당 Arena 할당 (CPU 수 * 4개 기본)
   스레드별 tcache(Thread Cache)로 락 경합 최소화

2. Size Class 세분화
   Small (8~14336 bytes): 섬세한 크기 분류
   Large (14336 bytes~): 개별 관리
   → 내부 단편화 최소화

3. 페이지 단위 메모리 반환
   사용하지 않는 페이지를 OS에 적극 반환 (munmap)
   ptmalloc보다 RSS와 실제 사용량의 차이가 작음

4. 단편화 통계 (Redis MEMORY DOCTOR)
   $ redis-cli MEMORY DOCTOR
   "Sam, I detected a memoryfragmentation problem ..."
   → jemalloc의 내부 통계를 Redis가 활용

Redis가 jemalloc을 선택한 이유:
  → 빈번한 다양한 크기의 키-값 할당/해제
  → ptmalloc 대비 단편화 감소 (실험적으로 20~30% 개선)
  → ACTIVE DEFRAG와 연동한 단편화 해소 가능
```

### tcmalloc (Google 개발, gRPC/Chrome 등)

```
tcmalloc의 핵심 특징:

1. 완전한 스레드 로컬 캐시
   각 스레드가 독립 캐시 보유 (락 없음)
   작은 할당(< 256KB): 스레드 캐시에서 직접 처리
   큰 할당: Central Cache → Page Heap

2. 극도로 빠른 작은 메모리 할당
   2~3 나노초 수준 (ptmalloc의 5~10배 빠름)

3. 정밀한 메모리 사용 통계
   gperftools의 heap profiler와 통합
   어떤 코드가 얼마나 할당했는지 추적 가능

적합한 워크로드:
  - 다수의 작은 객체를 빈번히 할당/해제
  - 멀티스레드 서버 (gRPC 서버 등)
  - CPU 바운드 + 메모리 할당이 병목인 경우
```

### JVM의 독자적 메모리 관리

```
JVM이 자체 Heap을 관리하는 이유:

1. GC 요구사항
   GC는 객체 이동(Compaction)이 필요
   → malloc()은 주소를 고정
   → JVM은 자유롭게 객체를 이동시킴으로써 단편화 제거

2. 예측 가능한 할당
   JVM의 Young Generation: Bump Pointer 할당
   → 현재 포인터를 객체 크기만큼 이동
   → O(1) 할당, 단편화 없음
   (대신 GC로 죽은 객체 회수)

3. OS 할당기 오버헤드 제거
   매번 malloc()/free() 대신
   대용량 mmap()으로 한번에 예약 → 내부적으로 관리

JVM Heap 구조 (G1GC):
  ┌─────────────────────────────────┐
  │           JVM Heap (mmap)       │
  │  ┌───────┬───────┬───────┐      │
  │  │Region │Region │Region │ ...  │  ← 1~32MB Region 단위
  │  │Young  │Old    │Humong.│      │
  │  └───────┴───────┴───────┘      │
  └─────────────────────────────────┘
  → Region 단위로 GC
  → 빈 Region은 다른 용도로 재사용 (단편화 없음)
```

---

## 💻 실전 실험

### 실험 1: Redis 단편화 확인 및 해소

```bash
# Docker로 Redis 실행
$ docker run -d --name redis-frag -p 6379:6379 redis:7

# 단편화 유발: 다양한 크기의 키-값 반복 생성/삭제
$ redis-cli --pipe << 'EOF'
$(python3 -c "
import random, string
for i in range(100000):
    size = random.randint(10, 10000)
    val = ''.join(random.choices(string.ascii_letters, k=size))
    print(f'SET key:{i} {val}')
")
EOF

# 절반 삭제
$ redis-cli --eval - << 'EOF'
for i = 1, 50000 do
  redis.call('DEL', 'key:' .. i)
end
return 'done'
EOF

# 단편화 확인
$ redis-cli info memory | grep -E "used_memory|fragmentation"
used_memory: 536870912         # 512MB (실제 데이터)
used_memory_rss: 858993459     # 819MB (OS 할당)
mem_fragmentation_ratio: 1.60  # 단편화 발생!

# Active Defrag 활성화
$ redis-cli config set activedefrag yes
$ sleep 30
$ redis-cli info memory | grep mem_fragmentation_ratio
mem_fragmentation_ratio: 1.08  # 개선됨
```

### 실험 2: Java NMT로 Native Memory 추적

```bash
# NMT 활성화 후 JVM 시작
$ java -XX:NativeMemoryTracking=detail -Xmx2g -jar app.jar &
$ PID=$!

# 기준선 스냅샷
$ jcmd $PID VM.native_memory baseline

# 부하 실행 후 변화 확인
$ sleep 60

$ jcmd $PID VM.native_memory detail.diff
[메모리 변화량 출력]
Native Memory Tracking:
Total: reserved=3200MB +200MB, committed=2100MB +100MB
- Java Heap (reserved=2048MB, committed=2048MB)
  (malloc=2048MB #1)
- Internal (reserved=50MB +50MB, committed=50MB +50MB)  ← 증가!
  (malloc=50MB +50MB #1000 +1000)
# Internal 영역 증가: Native 메모리 누수 의심
```

### 실험 3: 할당기별 성능 비교

```bash
# ptmalloc (기본)
$ time for i in $(seq 1 100000); do
    redis-cli set "key:$i" "$(head -c 100 /dev/urandom | base64)" > /dev/null
  done
real 45.3s

# jemalloc (Redis 기본)
$ docker run -d --name redis-jemalloc redis:7
$ time for i in $(seq 1 100000); do
    docker exec redis-jemalloc redis-cli set "key:$i" "$(head -c 100 /dev/urandom | base64)" > /dev/null
  done
real 38.7s  # 약 15% 빠름
```

---

## 📊 성능/비용 비교

| 할당기 | 단편화 | 스레드 확장성 | 메모리 반환 | 주요 사용처 |
|--------|-------|------------|-----------|-----------|
| ptmalloc | 높음 | 중간 (Arena) | 제한적 | glibc 기본 |
| jemalloc | 낮음 | 높음 | 적극적 | Redis, Firefox |
| tcmalloc | 낮음 | 매우 높음 | 중간 | gRPC, Chrome |
| JVM (G1) | 없음 | N/A (GC) | GC 시 | Java 애플리케이션 |

**Redis mem_fragmentation_ratio 해석**:

```
비율    의미                        권장 조치
< 1.0   메모리가 스왑됨              즉시 조사 필요
1.0~1.5 정상                        모니터링 유지
1.5~2.0 단편화 주의                 activedefrag 활성화
> 2.0   단편화 심각                 Redis 재시작 또는 activedefrag 강화
```

---

## ⚖️ 트레이드오프

**메모리 할당기 선택의 트레이드오프**

ptmalloc은 glibc에 내장되어 있어 별도 설치가 필요 없지만 단편화에 취약하다. jemalloc은 단편화를 줄이고 페이지 반환이 적극적이지만, 메모리 사용량 예측이 ptmalloc보다 복잡하다. tcmalloc은 멀티스레드에서 매우 빠르지만 메모리 오버헤드가 있다.

**Active Defrag의 비용**

Redis의 `activedefrag`는 단편화를 해소하지만 CPU를 추가로 사용한다. `active-defrag-cpu-pct`(기본 25%)로 CPU 사용률을 제한하면서 단편화 해소 속도를 조절할 수 있다. 단편화가 심하면 CPU를 더 써서라도 해소하는 편이 낫지만, CPU 여유가 없는 서버에서는 오히려 레이턴시가 증가할 수 있다.

---

## 📌 핵심 정리

```
malloc() = 커널 직접 호출 아님
  할당기가 큰 블록을 미리 확보 → 내부에서 분할 제공
  free() 즉시 OS 반환 안 함 → 내부 free list 재사용

메모리 단편화:
  외부 단편화: 연속된 빈 블록 부재
  내부 단편화: 블록 내 사용 안 하는 공간
  장기 실행 서버에서 누적 → RSS 증가

jemalloc (Redis):
  ptmalloc 대비 단편화 낮음
  페이지 단위 OS 반환 적극적
  mem_fragmentation_ratio로 모니터링

tcmalloc (gRPC 등):
  스레드 로컬 캐시로 락 경합 없음
  작은 할당에서 극도로 빠름

JVM:
  OS 할당기 사용 안 함
  Bump Pointer로 Young Gen 할당 (O(1), 단편화 없음)
  GC Compaction으로 단편화 자동 제거

핵심 진단:
  Redis: mem_fragmentation_ratio > 1.5 → activedefrag
  Java: jcmd VM.native_memory → Native Memory 영역별 추적
  OS: valgrind / gperftools → 할당 패턴 분석
```

---

## 🤔 생각해볼 문제

**Q1.** Redis `mem_fragmentation_ratio`가 0.8이다. 이것이 의미하는 바는?

<details>
<summary>해설 보기</summary>

`used_memory_rss < used_memory`로, Redis가 보고하는 데이터 크기보다 OS에서 할당받은 물리 메모리가 더 작다는 뜻이다. 이는 일반적으로 **스왑**이 발생했음을 의미한다. Redis 데이터의 일부가 디스크 스왑으로 밀려나 RSS로 집계되지 않는 것이다. 즉시 `vmstat`과 `/proc/$(pgrep redis)/status`의 `VmSwap`을 확인해야 한다. 스왑 사용이 확인되면 메모리를 증설하거나 `maxmemory` 설정을 낮춰야 한다.

</details>

**Q2.** Java 서비스에서 `ByteBuffer.allocateDirect()`로 Direct Buffer를 할당했다. 이 메모리는 JVM Heap에 포함되는가? 어떤 할당기를 사용하는가?

<details>
<summary>해설 보기</summary>

JVM Heap에 포함되지 않는다. Direct Buffer는 OS 레벨에서 `mmap()` 또는 `malloc()`으로 할당된 Native Memory 영역이다. JVM Heap의 GC 대상이 아니지만, `ByteBuffer` 객체가 GC로 수집될 때 `Cleaner`가 Native 메모리를 해제한다. 사용하는 할당기는 JVM 구현에 따라 다르지만 일반적으로 OS 기본 할당기(ptmalloc)를 사용한다. `jcmd VM.native_memory`의 `Internal` 항목에서 모니터링할 수 있으며, `-XX:MaxDirectMemorySize`로 한도를 설정할 수 있다.

</details>

**Q3.** 동일한 크기의 문자열 1000개를 Redis에 저장하고 전부 삭제했다. 이후 Redis 프로세스의 RSS는 저장 전으로 돌아가는가?

<details>
<summary>해설 보기</summary>

**즉시 돌아가지 않는다.** jemalloc은 해제된 메모리를 내부 free list에 보관하고, 페이지 단위로 묶인 메모리만 OS에 반환한다. 다양한 크기가 혼재된 페이지는 일부 블록이 살아있으면 반환하지 못한다. 단, jemalloc은 ptmalloc보다 적극적으로 비어있는 페이지를 OS에 반환한다. Active Defrag를 활성화하면 단편화된 메모리를 압축해 반환 가능한 페이지를 만든다. 동일 크기의 객체만 할당/해제하면 단편화가 거의 없어 비교적 빠르게 반환된다.

</details>

---

**[⬅️ 이전: mmap과 Direct I/O](./04-mmap-direct-io.md)** | **[홈으로 🏠](../README.md)** | **[다음: OOM Killer ➡️](./06-oom-killer.md)**
