# 02. 디스크 I/O 스택 — 애플리케이션에서 디바이스까지

## 🎯 핵심 질문

- `write()`를 호출하면 데이터가 디스크에 저장되기까지 어떤 레이어를 거치는가?
- 블록 레이어의 I/O 스케줄러는 왜 존재하고, 어떤 문제를 해결하는가?
- `iostat -xz 1`의 `await`, `%util`, `r/s`, `w/s`는 어느 레이어를 반영하는가?
- SSD 서버에서 I/O 스케줄러를 `none`으로 설정하는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**MySQL 서버에서 `iostat`의 `await`이 높다. 디스크가 느린 건가, 쿼리가 느린 건가?** `await`은 I/O 요청이 블록 레이어 큐에서 대기한 시간 + 실제 디스크 처리 시간의 합이다. `await`이 높은데 `%util`이 낮다면 I/O 스케줄러 큐 대기가 원인이다. `await`이 높고 `%util`도 높다면 실제 디스크 포화 상태다.

**Kafka 서버를 NVMe SSD로 교체했는데 I/O 스케줄러가 `mq-deadline`이었다.** NVMe SSD는 내부적으로 이미 큐 관리를 수행하므로 추가 소프트웨어 스케줄러가 오히려 레이턴시를 증가시킨다. `none`(또는 `noop`) 스케줄러로 변경하면 불필요한 레이어를 제거해 레이턴시가 감소한다.

**`dd if=/dev/zero of=/tmp/test bs=4k count=10000`과 `bs=1M count=40`은 같은 40MB를 쓰지만 속도가 다르다.** 블록 레이어의 병합(merge) 기능과 I/O 크기가 성능에 어떤 영향을 주는지 이해해야 한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: await만 보고 판단
$ iostat -xz 1
Device  r/s   w/s  rMB/s  wMB/s  await  %util
sda     0.5   150    0.0    2.3   45.2   92.1%

# "await 45ms면 느린 디스크" → 반은 맞고 반은 틀림
# await = 큐 대기 시간 + 디스크 처리 시간
# %util 92% → 디스크 포화 상태 (이게 진짜 문제)
# r_await vs w_await 분리 확인 필요 (iostat -x 옵션)

# 실수 2: HDD에 최적화된 스케줄러를 SSD에 적용
$ cat /sys/block/nvme0n1/queue/scheduler
[mq-deadline] kyber none
# NVMe SSD에 mq-deadline → 불필요한 큐 재정렬 오버헤드
# → none으로 변경해야 레이턴시 감소

# 실수 3: 작은 블록 크기로 대량 쓰기
$ dd if=/dev/zero of=/tmp/test bs=512 count=80000  # 총 40MB
# vs
$ dd if=/dev/zero of=/tmp/test bs=1M count=40  # 총 40MB
# 전자가 훨씬 느림 (512 * 80000 = 80000번의 write() vs 40번)
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# iostat 올바른 해석
$ iostat -xz 1 sda
Device  r/s   w/s  r_await  w_await  aqu-sz  %util
sda     1.0   200    3.2     45.2     12.5    95%

# r_await 3.2ms: 읽기 요청 대기 양호
# w_await 45ms: 쓰기 요청 대기 과도 → 쓰기 병목
# aqu-sz 12.5: 평균 큐 깊이 12.5개 → 포화 상태
# %util 95%: 디스크 거의 포화

# I/O 스케줄러 최적 설정
$ cat /sys/block/sda/queue/scheduler    # HDD
[mq-deadline] kyber none
# mq-deadline이 적합 (순서 보장, HDD 헤드 이동 최소화)

$ cat /sys/block/nvme0n1/queue/scheduler  # NVMe SSD
mq-deadline kyber [none]
# none 적합 (SSD는 자체 큐 관리, 소프트웨어 스케줄러 불필요)

# 변경 (재부팅 후 유지하려면 udev rule 필요)
$ echo none > /sys/block/nvme0n1/queue/scheduler

# 큐 깊이 조정 (SSD는 더 큰 값이 유리)
$ cat /sys/block/nvme0n1/queue/nr_requests
1024
$ echo 2048 > /sys/block/nvme0n1/queue/nr_requests
```

---

## 🔬 내부 동작 원리

### 디스크 I/O 전체 스택

```
[애플리케이션]
  write(fd, buf, 4096)
         │
         ▼
[시스템 콜 인터페이스]
  sys_write() → VFS write()
         │
         ▼
[VFS 레이어]
  file->f_op->write() 호출
  → 파일 시스템 write 함수로 분기
         │
         ▼
[파일 시스템 레이어] (ext4, XFS 등)
  → Page Cache에 데이터 기록 (Dirty 표시)
  → 메타데이터(inode) 업데이트
  → journal/WAL 처리 (ext4 journaling)
  → block_write_begin() → 블록 매핑 계산
         │
         ▼ (비동기 또는 fsync() 호출 시)
[블록 레이어]
  bio 구조체 생성 (Block I/O Request)
  → I/O 스케줄러 큐에 추가
  → 병합(merge): 인접 요청 합치기
  → 정렬(sort): HDD는 섹터 번호 순 정렬
  → 디바이스 드라이버에 전달
         │
         ▼
[장치 드라이버]
  → DMA 설정: 메모리→디바이스 직접 전송
  → 하드웨어 명령 전송 (SCSI, NVMe 명령어)
         │
         ▼
[물리 장치 (HDD/SSD)]
  → 실제 데이터 기록
  → 완료 인터럽트 발생
         │
         ▼ (인터럽트 핸들러)
[커널 완료 처리]
  → bio 완료 콜백
  → 대기 중인 프로세스 Wake-up (fsync 블로킹 해제)
```

### I/O 스케줄러의 역할

```
I/O 스케줄러가 없다면 (raw 순서):
  요청 1: 섹터 100
  요청 2: 섹터 5000
  요청 3: 섹터 200
  요청 4: 섹터 4800
  → HDD: 100 → 5000 → 200 → 4800 (헤드가 앞뒤로 왕복)
  → 총 이동 거리: (5000-100) + (5000-200) + (4800-200) = 14300

I/O 스케줄러 적용 후 (섹터 번호 정렬):
  100 → 200 → 4800 → 5000
  → 총 이동 거리: (200-100) + (4800-200) + (5000-4800) = 5000
  → 이동 거리 65% 감소!

병합(Merge) 동작:
  요청 A: 섹터 100~107 (4KB 쓰기)
  요청 B: 섹터 108~115 (4KB 쓰기, A 직후 인접!)
  → 병합: 섹터 100~115 (8KB 하나의 요청)
  → 시스템 콜 2번 → 디스크 요청 1번 (50% 감소)
```

### I/O 스케줄러 종류와 특성

```
1. CFQ (Completely Fair Queuing) — 레거시, 제거됨
   각 프로세스에 공평한 I/O 대역폭 할당
   단점: SSD에서 불필요한 오버헤드

2. deadline (mq-deadline — 최신 커널)
   읽기/쓰기 각각 최대 대기 시간 보장
   읽기 deadline: 500ms, 쓰기 deadline: 5s
   HDD + 혼합 I/O 워크로드에 적합
   MySQL, PostgreSQL 서버 권장

3. noop / none (최신 커널)
   FIFO + 병합만 수행, 재정렬 없음
   SSD/NVMe에 최적 (자체 내부 스케줄링)
   낮은 레이턴시, 오버헤드 최소

4. kyber
   두 개의 토큰 버킷으로 읽기/쓰기 QoS 제어
   NVMe SSD에서 레이턴시 제어 가능
   고성능 SSD + 읽기 레이턴시 우선시 환경

선택 기준:
  HDD:        mq-deadline (헤드 이동 최소화)
  SATA SSD:   mq-deadline 또는 none
  NVMe SSD:   none (커널 오버헤드 최소화)
  혼합 워크로드 + QoS 필요: kyber
```

### iostat 각 지표와 스택 레이어 매핑

```
$ iostat -xz 1

r/s, w/s:  초당 완료된 읽기/쓰기 요청 수
           → 블록 레이어에서 완료 처리된 bio 수

rMB/s, wMB/s: 초당 읽기/쓰기 처리량
           → 디바이스로 전달된 총 데이터량

await:     요청 제출부터 완료까지 평균 시간 (ms)
           = 큐 대기 시간 + 디바이스 처리 시간
           → 높으면: 큐 포화 또는 디바이스 느림

r_await, w_await: 읽기/쓰기 각각의 await
           → 읽기가 높으면: 읽기 집중 병목
           → 쓰기가 높으면: 쓰기 집중 병목

aqu-sz:    평균 I/O 큐 깊이
           → 1 미만: 디바이스 여유 있음
           → 1 이상: 큐에 요청 쌓임 (포화 가능성)

%util:     디바이스가 I/O 처리 중인 시간 비율
           → 100% = 포화 상태 (단, SSD는 병렬 처리로 100%여도 여유 있을 수 있음)
           → HDD에서는 100%가 확실한 포화

svctm (deprecated):
           실제 디바이스 서비스 시간 (await - 큐 대기)
           신뢰성 낮아 최신 iostat에서 제거됨
```

---

## 💻 실전 실험

### 실험 1: I/O 스케줄러별 성능 비교

```bash
# 현재 스케줄러 확인
$ cat /sys/block/sda/queue/scheduler
[mq-deadline] kyber none

# 동일 부하로 스케줄러별 성능 측정
for scheduler in mq-deadline kyber none; do
    echo $scheduler > /sys/block/sda/queue/scheduler
    echo "=== $scheduler ==="
    # 랜덤 4KB 읽기 1000번 (HDD에서 차이 명확)
    fio --name=test --ioengine=sync --rw=randread \
        --bs=4k --numjobs=1 --size=1g \
        --filename=/dev/sda --direct=1 \
        --runtime=10 --time_based
done
# HDD: mq-deadline > kyber > none (정렬 효과)
# NVMe: none > kyber > mq-deadline (오버헤드 차이)
```

### 실험 2: blktrace로 I/O 스택 실시간 추적

```bash
# blktrace: 블록 레이어 I/O 요청 추적
$ apt-get install -y blktrace

# MySQL I/O 패턴 추적 (10초)
$ blktrace -d /dev/sda -o /tmp/mysql_io &
$ sleep 10 && kill %1

# 분석: 병합 효과 확인
$ blkparse /tmp/mysql_io.blktrace.* | \
  awk '/^[0-9]/{print $6}' | sort | uniq -c
# Q: 큐에 추가 (커널이 받은 요청)
# M: 병합 (인접 요청 합침)
# D: 디바이스로 전달
# C: 완료

# Q 대비 D 수가 적으면 = 병합이 많이 발생 = 효율적
```

### 실험 3: iostat로 MySQL I/O 병목 진단

```bash
# MySQL 부하 중 iostat 모니터링
$ iostat -xz 1 sda 2>&1 | tee /tmp/iostat_mysql.log &

# MySQL 대량 INSERT 실행
$ mysql -e "INSERT INTO big_table SELECT ... FROM ..." &

# iostat 분석
$ awk '/sda/ {
    if ($14 > 80) print "⚠️  높은 %util:", $14"%"
    if ($10 > 20) print "⚠️  높은 await:", $10"ms"
    if ($6 > 1)   print "⚠️  높은 aqu-sz:", $6
}' /tmp/iostat_mysql.log

# 병합 효율 확인
$ cat /sys/block/sda/queue/stat
# 형식: reads_completed reads_merged reads_sectors ...
# merged / completed 비율이 높으면 병합 효과 있음
```

---

## 📊 성능/비용 비교

**I/O 스케줄러별 특성 비교 (1만 랜덤 읽기 기준)**:

```
HDD (7200RPM):
  none:         IOPS 80,   Latency 12ms  ← 정렬 없음, 느림
  mq-deadline:  IOPS 180,  Latency  5ms  ← 정렬 효과
  kyber:        IOPS 160,  Latency  6ms

NVMe SSD:
  none:         IOPS 580K, Latency 0.17ms  ← 최적
  mq-deadline:  IOPS 520K, Latency 0.19ms  ← 스케줄러 오버헤드
  kyber:        IOPS 550K, Latency 0.18ms
```

**블록 크기별 쓰기 처리량 (ext4, SSD)**:

```
bs=512B:   ~100 MB/s  (시스템 콜 오버헤드 지배)
bs=4KB:    ~400 MB/s  (일반적인 페이지 크기)
bs=64KB:   ~900 MB/s  (병합 효과 최대화)
bs=1MB:    ~1100 MB/s (대역폭 한계 근접)
bs=4MB:    ~1100 MB/s (포화 상태, 이득 없음)
→ 4KB~64KB: 실용적 최적 범위
```

---

## ⚖️ 트레이드오프

**I/O 스케줄러 없음(none)의 장단점**

SSD/NVMe에서 `none` 스케줄러는 커널 레벨 큐 관리를 생략해 레이턴시를 줄인다. SSD는 어느 섹터에나 동등한 속도로 접근하고 내부적으로 자체 큐를 관리하므로 커널 정렬이 불필요하다. 단, 여러 프로세스가 동시에 I/O를 발생시키는 환경에서 공정성 제어가 없어 특정 프로세스가 I/O 대역폭을 독점할 수 있다. cgroups I/O 제한으로 보완할 수 있다.

**병합의 효과와 레이턴시 트레이드오프**

I/O 스케줄러가 요청을 병합하면 처리량이 증가하지만, 병합을 위해 요청을 잠시 대기시키는 시간(I/O latency)이 생긴다. 실시간 게임 서버처럼 레이턴시가 절대적인 환경에서는 병합 없이 즉시 전달하는 것이 유리하다. 데이터베이스처럼 처리량이 중요한 환경에서는 병합의 이점이 크다.

---

## 📌 핵심 정리

```
디스크 I/O 스택 (위→아래):
  애플리케이션 → VFS → 파일 시스템 → 블록 레이어 → 장치 드라이버 → 디스크

블록 레이어 핵심:
  병합(Merge): 인접 요청 합침 → 시스템 콜 수 감소
  정렬(Sort):  HDD 섹터 순 정렬 → 헤드 이동 최소화

I/O 스케줄러 선택:
  HDD:   mq-deadline (정렬 효과 큼)
  SSD:   mq-deadline 또는 none
  NVMe:  none (커널 오버헤드 최소화)

iostat 핵심 지표:
  await  = 큐 대기 + 디바이스 처리 시간 (총 레이턴시)
  r_await/w_await = 읽기/쓰기 개별 레이턴시
  aqu-sz = 큐 깊이 (>1이면 포화 가능성)
  %util  = 디바이스 포화도 (HDD 100%=포화, SSD는 다름)

핵심 진단 명령어:
  iostat -xz 1           → I/O 스택 전체 현황
  cat /sys/block/*/queue/scheduler → 현재 I/O 스케줄러
  blktrace + blkparse    → 블록 레이어 요청 추적
  cat /sys/block/*/queue/stat → 병합 횟수 확인
```

---

## 🤔 생각해볼 문제

**Q1.** MySQL 서버에서 `iostat`의 `w/s`(초당 쓰기 수)가 1000이다. 그런데 MySQL의 `Com_insert`는 초당 100이다. 이 차이는 어떻게 설명할 수 있는가?

<details>
<summary>해설 보기</summary>

MySQL은 단일 INSERT 명령에도 여러 번의 디스크 쓰기가 발생한다. redo log(WAL) 쓰기, binlog 쓰기, InnoDB 데이터 페이지 쓰기, 시스템 테이블스페이스 쓰기 등이 한 트랜잭션에서 발생한다. 또한 Double Write Buffer는 데이터를 두 번 쓰고, InnoDB 체크포인트가 주기적으로 Dirty 페이지를 플러시한다. 실제로 `w/s = 1000`은 100 INSERT에 대해 평균 10번의 블록 레이어 쓰기가 발생한 것이며, 이는 정상적인 InnoDB 동작이다.

</details>

**Q2.** `dd if=/dev/zero of=/tmp/test bs=4k count=1 oflag=direct`와 일반 `dd`(direct 없음)의 차이는 무엇인가? 블록 레이어 관점에서 설명하라.

<details>
<summary>해설 보기</summary>

`oflag=direct`는 `O_DIRECT` 플래그를 사용해 Page Cache를 우회하고 블록 레이어로 직접 요청을 보낸다. 일반 `dd`는 Page Cache에 쓰고 즉시 반환하므로 실제 디스크 쓰기는 나중에 비동기로 발생한다. `O_DIRECT`는 즉시 블록 레이어 → 디바이스 드라이버 → 디스크 경로로 전달하므로 완료까지 더 오래 걸린다. 대신 Page Cache 메모리를 사용하지 않고, 진짜 디스크 성능을 측정할 수 있다. `fio`의 `--direct=1` 옵션도 같은 원리다.

</details>

**Q3.** `%util`이 100%인 SSD에서도 더 많은 I/O를 처리할 수 있다고 한다. 왜인가?

<details>
<summary>해설 보기</summary>

`%util`은 디바이스에 최소 하나의 I/O 요청이 처리 중인 시간 비율이다. HDD는 단일 헤드로 한 번에 하나의 I/O를 처리하므로 `%util=100%`가 진짜 포화다. SSD는 내부에 수십~수백 개의 병렬 채널(NAND 플래시 병렬성)을 가지고, NVMe는 최대 65,535개의 큐를 지원한다. 따라서 외부에서 보기에 `%util=100%`여도 내부적으로 병렬 처리 여유가 있다. 실제 포화 여부는 `await`과 `aqu-sz`의 증가 여부로 판단해야 한다.

</details>

---

**[⬅️ 이전: VFS](./01-vfs.md)** | **[홈으로 🏠](../README.md)** | **[다음: Sequential vs Random I/O ➡️](./03-sequential-vs-random-io.md)**
