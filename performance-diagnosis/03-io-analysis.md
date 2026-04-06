# 03. I/O 분석 — await와 %util로 디스크 병목 찾기

## 🎯 핵심 질문

- `iostat`의 `await`, `r_await`, `w_await`, `%util`은 각각 무엇을 의미하는가?
- SSD에서 `%util=100%`이지만 처리량이 남는 이유는?
- `iotop`으로 특정 프로세스의 I/O 병목을 어떻게 찾는가?
- MySQL과 Redis에서 I/O 병목이 발생하는 각각의 시나리오와 진단 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**MySQL 쿼리 응답이 갑자기 느려졌는데 CPU는 정상이다.** `iostat`에서 `await`이 50ms로 높다. 디스크 I/O 대기가 쿼리 응답 시간을 지배하는 것이다. 어느 디스크인지, 읽기인지 쓰기인지 `r_await`/`w_await`으로 구분한다.

**Redis `BGSAVE` 중에 레이턴시 스파이크가 발생한다.** `iotop`으로 확인하면 `redis-server` 자식 프로세스가 대량의 쓰기 I/O를 유발한다. `iostat`에서 `wMB/s`가 급증하고 `w_await`이 높아진다. RDB 파일 저장 경로를 NVMe 디스크로 이동하거나 `save` 빈도를 낮춰 해결한다.

**Kafka 브로커에서 Consumer lag이 갑자기 커진다.** 디스크 쓰기 처리량이 Consumer 읽기 처리량을 따라가지 못하는 것이다. `iostat`에서 `wMB/s`가 한계에 달했을 때 이 현상이 발생한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: %util만 보고 SSD 포화 판단
$ iostat -xz 1
Device  %util
nvme0n1  100%

# "NVMe가 100% 사용 중! 포화됐다!" → 잘못된 판단
# SSD는 내부 병렬 채널로 동시 처리 가능
# %util=100%이지만 추가 처리량 여유 있을 수 있음
# await와 aqu-sz로 실제 포화 판단해야 함

# 실수 2: iostat 해석 없이 디스크 교체 결정
# "await 20ms, 디스크 바꾸자" → 근거 없는 결정
# HDD: await 20ms = 정상
# SSD: await 20ms = 매우 높음
# 디바이스 특성을 모르고 단순 수치 비교

# 실수 3: 프로세스별 I/O 모름
# "디스크가 느리다" → 어느 프로세스 때문인지 모름
# iotop 없이 단순 iostat만 보면 원인 프로세스 특정 불가
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# I/O 병목 진단 흐름

# Step 1: 전체 디스크 현황
$ iostat -xz 1
# await 높음? %util 높음? 어느 디바이스?

# Step 2: 읽기/쓰기 구분
$ iostat -xz 1
# r_await vs w_await 비교
# rMB/s vs wMB/s 비교

# Step 3: 어느 프로세스가 I/O 유발?
$ iotop -bo -d 1
# 상위 I/O 프로세스 확인

# Step 4: 해당 프로세스 시스템 콜 확인
$ strace -e trace=read,write,pread,pwrite -p $PID 2>&1 | head -30

# Step 5: 파일 단위 확인
$ lsof -p $PID | grep -v "mem\|txt\|cwd\|rtd"
# 어떤 파일에 I/O하는지
```

---

## 🔬 내부 동작 원리

### iostat -xz 1 각 컬럼 완전 해석

```bash
$ iostat -xz 1

Device  r/s  w/s  rMB/s  wMB/s  r_await  w_await  aqu-sz  %util
sda     5.0  200   0.1    3.2     3.5      48.2     12.5    92%
nvme0   100  500  12.5   64.0    0.2       0.5      1.2     85%

# 컬럼별 의미:
r/s, w/s:    초당 완료된 읽기/쓰기 요청 수
             → 요청 빈도 (IOPS)

rMB/s, wMB/s: 초당 처리된 읽기/쓰기 데이터량
              → 처리량 (Throughput)

r_await:     읽기 요청의 평균 응답 시간 (ms)
             = I/O 스케줄러 큐 대기 + 디바이스 처리
             
w_await:     쓰기 요청의 평균 응답 시간 (ms)

aqu-sz:      평균 I/O 큐 깊이 (요청 수)
             → 0~1: 여유 있음
             → 1~4: 약간 바쁨
             → 4+:  포화 가능성

%util:       디바이스가 I/O 처리 중인 시간 비율
             → HDD: 70%+ = 포화 접근
             → SSD/NVMe: 100%여도 여유 있을 수 있음

svctm (제거됨): 순수 디바이스 처리 시간 (신뢰성 낮아 삭제)
```

### HDD vs SSD에서 %util 해석 차이

```
HDD 동작:
  단일 헤드 → 한 번에 하나의 I/O
  %util=100% = 디바이스 계속 작동 중
  → 추가 I/O 수용 불가 = 진짜 포화

  HDD 포화 판단:
    %util > 70% → 주의
    %util > 90% → 포화
    r/s + w/s 이상으로 증가해도 await 급증

NVMe SSD 동작:
  내부 병렬 채널 64개+
  I/O 큐 깊이 65535까지 지원
  %util = "디바이스에 최소 1개 I/O 처리 중인 시간"
  → 1개만 처리해도 %util=100% 될 수 있음!

  NVMe 실제 포화 판단:
    await 증가 (0.1ms → 1ms+)
    aqu-sz 증가 (1 → 10+)
    → 내부 큐가 가득 차기 시작할 때

예시:
  NVMe: r/s=100K, %util=100%, await=0.15ms → 여유 있음
  NVMe: r/s=100K, %util=100%, await=2ms   → 포화 접근
```

### iotop으로 프로세스별 I/O 추적

```bash
$ iotop -bo -d 1
# -b: 배치 모드, -o: I/O 있는 프로세스만, -d 1: 1초 간격

Total DISK READ:  12.50 MB/s | Total DISK WRITE: 64.00 MB/s
TID   PRIO USER  DISK READ  DISK WRITE   SWAPIN   IO>   COMMAND
1234  be/4 mysql    0.00 B  48.00 MB/s   0.00 %  95.0% mysqld
5678  be/4 kafka  12.50 MB/s  8.00 MB/s  0.00 %  35.0% java (kafka)
9012  be/4 redis    0.00 B  8.00 MB/s   0.00 %  15.0% redis-server

# TID: 스레드 ID
# PRIO: I/O 우선순위 (be=best effort, rt=real-time)
# IO%: I/O 대기 시간 비율
# → mysql이 48MB/s 쓰기 + 95% I/O 대기 = MySQL이 디스크 병목 원인

# 특정 프로세스만 모니터링
$ iotop -bo -p $MYSQL_PID
```

### MySQL I/O 병목 시나리오

```bash
# 시나리오 1: InnoDB Buffer Pool flush
$ iostat -xz 1  # wMB/s 급증, w_await 높음
# 원인: Dirty 페이지 flush (checkpoint)
# 확인:
$ mysql -e "SHOW ENGINE INNODB STATUS\G" | grep -A 5 "LOG"
# Log sequence number vs Log flushed up to 차이 확인

# 시나리오 2: 쿼리가 Index 없어 Full Scan
$ iostat -xz 1  # rMB/s 급증 (대량 읽기)
$ iotop -bo     # mysqld가 상단에 위치
# 확인:
$ mysql -e "SHOW PROCESSLIST"  # 느린 쿼리
$ mysql -e "EXPLAIN SELECT ..."  # 인덱스 사용 여부

# 시나리오 3: Binary Log 과다 생성
$ iostat -xz 1  # 특정 파티션 wMB/s 높음
$ df -h         # binlog 경로 파티션
$ ls -lh /var/lib/mysql/mysql-bin.* | tail -5
# binlog 크기 급증 확인
```

---

## 💻 실전 실험

### 실험 1: Sequential vs Random I/O 차이 측정

```bash
# fio로 정밀 측정
$ fio --name=seq_read --rw=read --bs=1M \
  --size=4G --direct=1 --ioengine=libaio --iodepth=32 \
  --filename=/dev/sdb --runtime=30 --time_based

# iostat로 동시 모니터링 (별도 터미널)
$ iostat -xz 1 sdb
# Sequential: rMB/s 높음, r/s 낮음 (큰 블록 읽기)

$ fio --name=rand_read --rw=randread --bs=4k \
  --size=4G --direct=1 --ioengine=libaio --iodepth=32 \
  --filename=/dev/sdb --runtime=30 --time_based

# Random: r/s 높음, rMB/s 낮음 (작은 블록 많이)
# HDD: await 급증 (헤드 탐색)
# NVMe: await 거의 차이 없음 (랜덤 접근 빠름)
```

### 실험 2: MySQL I/O 패턴 진단

```bash
# MySQL 부하 중 I/O 모니터링
$ sysbench oltp_read_write \
  --mysql-host=127.0.0.1 --db-driver=mysql \
  --tables=10 --table-size=1000000 \
  --threads=16 --time=60 run &

# iostat + iotop 동시 실행
$ iostat -xz 1 | grep -E "Device|sda|nvme" &
$ iotop -bo -d 1 | head -10 &

# 결과 해석:
# r_await > w_await: 읽기 병목 (Index Scan, Buffer Pool miss)
# w_await > r_await: 쓰기 병목 (flush, binlog)
# %util 낮은데 await 높음: I/O 스케줄러 큐 대기 (iowait 아닌 소프트웨어 병목)
```

### 실험 3: blkstat으로 I/O 분포 확인

```bash
# 블록 레이어 I/O 크기 분포 확인
$ cat /sys/block/sda/queue/iostats
# 또는
$ grep "" /sys/block/sda/stat

# I/O 크기 히스토그램 (최신 커널)
$ cat /sys/block/sda/queue/io_stat
# 크기별 I/O 분포: 4K, 8K, 16K, ... 각각 몇 건

# 실제 I/O 크기가 애플리케이션 의도와 맞는지 확인
# MySQL InnoDB: 16KB 페이지 크기
$ cat /sys/block/sda/stat | awk '{print "read_sectors:", $3, "write_sectors:", $7}'
# 읽기 섹터 / 읽기 횟수 = 평균 읽기 크기
```

---

## 📊 성능/비용 비교

**디바이스 유형별 정상 await 범위**:

```
디바이스        정상 r_await   높음 기준   극심함
HDD (7200RPM)    1~8ms        > 20ms      > 50ms
SATA SSD         0.1~0.5ms    > 2ms       > 10ms
NVMe SSD (PCIe4) 0.02~0.1ms  > 0.5ms     > 2ms
배터리 RAID       < 0.1ms     > 0.5ms     > 1ms
```

**%util과 aqu-sz 조합 해석**:

```
aqu-sz  %util  해석
< 1     < 50%  디바이스 여유 충분
1~4     50~80% 적당히 사용 중
4~8     80~99% 포화 시작 (HDD는 심각)
> 8     ~100%  포화 (SSD도 문제 가능성)
```

---

## ⚖️ 트레이드오프

**I/O 스케줄러 선택 (Ch4에서 다뤘지만 진단 관점)**

진단 중 `aqu-sz`가 높고 `await`도 높지만 `%util`이 낮다면, 디바이스 자체 포화가 아니라 소프트웨어 I/O 스케줄러 큐에서 재정렬을 기다리는 것이다. NVMe를 사용하면서 `mq-deadline` 스케줄러를 쓰면 이런 현상이 발생한다. `none` 스케줄러로 변경하면 개선된다.

**O_DIRECT의 영향**

MySQL `innodb_flush_method=O_DIRECT`는 Page Cache를 우회해 iostat에 직접적인 I/O가 보인다. `fsync` 모드에서는 Page Cache가 버퍼 역할을 해 iostat에서는 더 작은 I/O처럼 보일 수 있다. 진단 시 `innodb_flush_method` 설정에 따라 iostat 해석이 달라질 수 있다.

---

## 📌 핵심 정리

```
iostat 핵심 지표:
  await    = 전체 I/O 응답 시간 (큐 대기 + 처리)
  r_await  = 읽기 응답 시간
  w_await  = 쓰기 응답 시간
  aqu-sz   = 평균 큐 깊이 (1 이상이면 대기 있음)
  %util    = 처리 중인 시간 비율 (SSD에서 오해 주의)

HDD vs SSD %util 해석:
  HDD: 70%+ = 포화, 100% = 완전 포화
  SSD: 100%여도 처리량 여유 있을 수 있음
  → await와 aqu-sz가 실제 포화 지표

iotop 활용:
  -b -o -d 1 : 배치, I/O있는것만, 1초간격
  → 어느 프로세스가 I/O 병목 원인인지 특정

MySQL I/O 병목 패턴:
  r_await 높음 → Buffer Pool miss, Full Scan
  w_await 높음 → checkpoint flush, binlog
  iotop에서 mysqld TID별 확인

Redis I/O 병목 패턴:
  BGSAVE/BGREWRITEAOF 중 w_await 급증
  RDB/AOF 파일 경로를 빠른 디스크로 이동

핵심 명령어:
  iostat -xz 1           → 디스크 I/O 통계
  iotop -bo -d 1         → 프로세스별 I/O
  strace -e pread,pwrite → I/O 시스템 콜 추적
  lsof -p <pid>          → 어떤 파일에 I/O?
```

---

## 🤔 생각해볼 문제

**Q1.** `iostat`에서 `r/s=0`, `rMB/s=0`, `w/s=5`, `wMB/s=0.1`이다. `w_await=200ms`다. 무엇이 문제인가?

<details>
<summary>해설 보기</summary>

초당 5번의 쓰기 요청이 있는데 각 요청이 평균 200ms를 기다린다. `wMB/s=0.1`이면 요청당 20KB 정도의 작은 쓰기다. 쓰기 요청 수가 많지 않은데 대기 시간이 길다는 것은 파일 시스템 레벨 동기화(fsync) 때문일 가능성이 높다. MySQL `innodb_flush_log_at_trx_commit=1`이나 Redis `appendfsync=always` 설정에서 발생하는 패턴이다. `strace -e trace=fsync -p <pid>`로 fsync 호출 빈도를 확인하고, 내구성 설정을 검토한다.

</details>

**Q2.** NVMe SSD에서 `%util=100%`, `await=0.15ms`, `aqu-sz=2`다. 디스크를 교체해야 하는가?

<details>
<summary>해설 보기</summary>

교체 필요 없다. NVMe에서 `await=0.15ms`는 매우 건강한 수치다(NVMe 정상 범위: 0.02~0.1ms). `aqu-sz=2`는 약간의 큐 깊이가 있지만 문제 수준이 아니다. `%util=100%`은 NVMe가 지속적으로 사용되는 것을 나타내지만 포화를 의미하지 않는다. 처리량이 더 필요하다면 `aqu-sz`를 늘리거나(`fio --iodepth=64`) 병렬 요청을 늘리면 된다. 실제 포화는 `await`이 1ms를 넘고 `aqu-sz`가 수십에 달할 때다.

</details>

**Q3.** `iotop`에서 MySQL의 TID가 여러 개 보인다. 어느 TID의 I/O가 병목인지 어떻게 특정하는가?

<details>
<summary>해설 보기</summary>

`iotop -bo -d 1`에서 `IO%`가 가장 높은 TID를 찾는다. 해당 TID의 `strace -p <tid>`로 어떤 시스템 콜을 하는지 확인한다(`pread64`, `pwrite64`, `fsync` 등). MySQL Performance Schema에서 `events_waits_current`나 `events_io_summary_by_thread_by_event_name` 테이블을 조회하면 스레드별 I/O 대기 시간을 MySQL 레벨에서 확인할 수 있다. `SHOW PROCESSLIST`와 TID를 연결해 어떤 쿼리가 해당 I/O를 유발하는지 추적한다.

</details>

---

**[⬅️ 이전: 메모리 분석](./02-memory-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: 네트워크 분석 ➡️](./04-network-analysis.md)**
