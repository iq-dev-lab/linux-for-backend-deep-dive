# 03. Sequential vs Random I/O — HDD와 SSD의 진실

## 🎯 핵심 질문

- HDD에서 Sequential I/O가 Random I/O보다 수백 배 빠른 물리적 이유는?
- SSD는 랜덤 접근이 빠른데도 왜 Sequential I/O가 더 유리한가?
- MySQL InnoDB가 랜덤 쓰기를 Sequential 쓰기로 변환하는 원리는?
- Kafka가 오직 Sequential Append 설계를 고집하는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Kafka 토픽에 파티션 수를 늘렸더니 처리량이 오히려 줄었다.** 파티션이 많아지면 각 파티션의 세그먼트 파일에 번갈아 쓰면서 Random I/O로 전환된다. Kafka의 핵심 설계 원칙인 Sequential Append가 깨지는 것이다. 파티션 수는 처리량과 I/O 패턴 사이의 트레이드오프다.

**MySQL에서 `innodb_buffer_pool_size`를 늘렸더니 Dirty 페이지 플러시 시 I/O 스파이크가 줄었다.** Buffer Pool이 클수록 Dirty 페이지를 모아서 Sequential로 플러시하는 효율이 높아진다. I/O 패턴이 Sequential에 가까워져 디스크 부담이 감소한다.

**Elasticsearch 인덱스 샤드 크기가 클수록 머지(segment merge) I/O가 효율적이다.** Lucene 세그먼트 머지는 여러 작은 세그먼트를 Sequential 읽기 + Sequential 쓰기로 합친다. 샤드가 너무 작으면 머지 대상이 분산되어 Random I/O 패턴이 된다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: SSD니까 Random I/O도 무조건 빠르다
# SSD 랜덤 읽기: ~500K IOPS (NVMe 기준)
# SSD 순차 읽기: ~3000 MB/s = ~750K IOPS (4KB 기준)
# → 여전히 Sequential이 약 1.5배 빠름
# → 쓰기 증폭 고려하면 더 차이남

# 실수 2: Kafka 파티션 수를 무조건 늘림
$ kafka-topics.sh --alter --topic my-topic --partitions 100
# 파티션 100개 → 동시에 100개 파일에 순환 쓰기
# → 각 파일 단독으로는 Sequential이지만
# → 디스크 입장에서는 100개 파일 간 Random I/O
# → 처리량 감소 가능

# 실수 3: MySQL에서 랜덤 I/O 패턴을 줄이려는 시도 없이 디스크 업그레이드만
# → Double Write Buffer 원리 이해 없이 무조건 하드웨어 투자
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# I/O 패턴 측정 (Sequential vs Random 비율)
$ fio --name=seq_read --ioengine=sync --rw=read \
  --bs=64k --size=1g --direct=1 --filename=/dev/sda
# Sequential Read: ~200 MB/s (HDD 기준)

$ fio --name=rand_read --ioengine=sync --rw=randread \
  --bs=4k --size=1g --direct=1 --filename=/dev/sda
# Random Read: ~0.8 MB/s (HDD 기준) ← 250배 차이!

# iostat로 실시간 I/O 패턴 확인
$ iostat -xz 1
# r/s 낮고 rMB/s 높음 → Sequential (큰 I/O, 요청 수 적음)
# r/s 높고 rMB/s 낮음 → Random (작은 I/O, 요청 수 많음)

# Kafka 최적 파티션 수 산정
# 처리량(MB/s) / 디스크당 Sequential 쓰기 속도 = 최소 파티션 수
# 예: 1000MB/s 처리, 디스크 500MB/s Sequential → 최소 2 파티션
```

---

## 🔬 내부 동작 원리

### HDD에서 Sequential vs Random의 물리적 차이

```
HDD 구조:
  ┌─────────────────────────────────┐
  │  플래터 (자성 디스크, 여러 장)        │
  │    ┌────────────────────┐       │
  │    │  트랙 (동심원)        │       │
  │    │   ├── 섹터 (512B)   │       │
  │    │   ├── 섹터          │       │
  │    └────────────────────┘       │
  │  헤드 (읽기/쓰기 암)                │
  └─────────────────────────────────┘

Sequential I/O:
  헤드가 플래터 위를 이동 없이 회전하며 읽기
  → 회전 대기만 (Rotational Latency): ~2~4ms (7200RPM)
  → 처리량: ~100~200 MB/s

Random I/O:
  요청마다 헤드가 다른 트랙으로 이동 (탐색)
  → Seek Time: ~5~10ms (헤드 이동)
  → Rotational Latency: ~2~4ms
  → 합계: ~7~14ms per I/O
  → 처리량: 4KB 랜덤 = 0.5~1 MB/s (150~200배 차이!)

결론: HDD에서 Sequential과 Random의 성능 차이는 헤드 탐색 비용
  → 이것이 Kafka, MySQL 등이 Sequential I/O를 설계 원칙으로 삼는 이유
```

### SSD에서도 Sequential이 유리한 이유

```
SSD 내부 구조:
  [컨트롤러]
    ├── 채널 0 → [NAND 패키지 0-1]
    ├── 채널 1 → [NAND 패키지 2-3]
    ├── 채널 2 → [NAND 패키지 4-5]
    └── 채널 3 → [NAND 패키지 6-7]

NAND 플래시 읽기/쓰기 단위:
  페이지(Page): 4KB~16KB (읽기/쓰기 단위)
  블록(Block): 512KB~4MB (삭제 단위)

SSD Random I/O의 비용:
  1. 랜덤 4KB 쓰기 → 별도 페이지에 각각 기록
  2. 기존 블록의 해당 페이지 무효화
  3. Write Amplification(쓰기 증폭):
     블록(예: 1MB) 전체를 지워야 새 데이터 쓰기 가능
     → 4KB 쓰기가 결국 1MB 블록 삭제 유발

Sequential I/O의 이점:
  연속 페이지 쓰기 → 동일 블록 내에서 순서대로 기록
  → 블록 삭제 횟수 최소화
  → Write Amplification 감소
  → SSD 수명 연장 + 처리량 향상

쓰기 증폭 비교:
  완전 Sequential: 쓰기 증폭 ≈ 1 (이상적)
  완전 Random:     쓰기 증폭 ≈ 블록크기/페이지크기 = 수십~수백
```

### MySQL InnoDB: Random → Sequential 변환

```
InnoDB Random 쓰기 문제:
  UPDATE users SET name='...' WHERE id=12345
  → users 테이블의 12345번 페이지 수정
  → 이 페이지는 디스크의 임의 위치
  → 수천 개의 UPDATE = 수천 번의 Random Write

Double Write Buffer (이중 쓰기 버퍼):
  목적 1: Random → Sequential 변환
  목적 2: 파셜 페이지 쓰기(Partial Page Write) 방지

  동작 과정:
  ① Dirty 페이지들을 Buffer Pool에서 수집 (여러 페이지)
  ② Double Write Buffer 파일(ibdata1)에 Sequential로 먼저 쓰기
     (연속 쓰기 = Sequential I/O!)
  ③ 각 페이지를 실제 데이터 파일 위치에 쓰기 (Random I/O)
  ④ Double Write Buffer 엔트리 삭제

  Sequential 효과:
  ② 단계에서 100개 페이지를 Sequential로 한번에 쓰기
  → 100번의 Random Write 전에 checkpoint 역할
  → 전체 I/O 패턴이 Sequential 비율 증가

  innodb_doublewrite = OFF 시:
  → Double Write 없음 (약 5~10% 성능 향상)
  → 파셜 페이지 쓰기 위험 (전원 장애 시 데이터 손상 가능)
  → 신뢰성 높은 저장소(배터리 지원 RAID)에서만 권장
```

### Kafka의 Sequential Append 설계

```
Kafka 파티션 파일 구조:
  /kafka/data/my-topic-0/
    ├── 00000000000000000000.log   ← 세그먼트 파일 (Sequential Append)
    ├── 00000000000000000000.index ← 오프셋 인덱스
    ├── 00000000001234567890.log   ← 다음 세그먼트
    └── ...

Producer 쓰기:
  append(message) → 현재 세그먼트 파일 끝에 추가
  = 완전 Sequential Write (항상 파일 끝)

Consumer 읽기:
  특정 오프셋부터 읽기
  → 대부분 최신 메시지 = Page Cache 히트 (디스크 없음)
  → 오래된 메시지 = Sequential Read (앞에서 뒤로)

Zero-copy + Sequential I/O 조합:
  Page Cache에서 sendfile() → NIC 직접 전송
  = 디스크 읽기 없음 + 복사 없음 = 최고 성능

파티션 수와 Sequential I/O 트레이드오프:
  파티션 1개: 완전 Sequential → 디스크 최대 처리량
  파티션 N개: N개 파일에 라운드로빈 → 사실상 Random I/O 분산
  권장: 디스크당 파티션 수 = 디스크 처리량 / 메시지 처리량으로 계산
```

---

## 💻 실전 실험

### 실험 1: HDD Sequential vs Random I/O 성능 측정

```bash
# fio로 정밀 측정 (Direct I/O로 Page Cache 제거)
# Sequential Read
$ fio --name=seq_read --ioengine=libaio --rw=read \
  --bs=64k --iodepth=32 --size=4g \
  --filename=/dev/sdb --direct=1 --runtime=30
# Sequential Write
$ fio --name=seq_write --ioengine=libaio --rw=write \
  --bs=64k --iodepth=32 --size=4g \
  --filename=/dev/sdb --direct=1 --runtime=30

# Random Read (4KB)
$ fio --name=rand_read --ioengine=libaio --rw=randread \
  --bs=4k --iodepth=32 --size=4g \
  --filename=/dev/sdb --direct=1 --runtime=30
# Random Write (4KB)
$ fio --name=rand_write --ioengine=libaio --rw=randwrite \
  --bs=4k --iodepth=32 --size=4g \
  --filename=/dev/sdb --direct=1 --runtime=30

# 예상 결과 (HDD):
# seq_read:  200 MB/s  / rand_read:  1 MB/s  (200배 차이)
# seq_write: 180 MB/s  / rand_write: 0.8 MB/s
```

### 실험 2: MySQL Double Write Buffer 효과 측정

```bash
# Double Write 활성화 (기본)
$ mysql -e "SHOW VARIABLES LIKE 'innodb_doublewrite';"
innodb_doublewrite: ON

# sysbench로 OLTP 워크로드 측정
$ sysbench oltp_write_only \
  --mysql-host=127.0.0.1 --db-driver=mysql \
  --tables=10 --table-size=1000000 \
  --threads=16 --time=60 run

# iostat에서 쓰기 패턴 확인
$ iostat -xz 1 | grep -E "sda|Device"
# Double Write ON: w/s 높지만 wMB/s도 상대적으로 높음 (Sequential)
# Double Write OFF: w/s 더 낮지만 랜덤 I/O 비율 높음

# 파셜 쓰기 실험 (실제 운영에서 절대 하면 안 됨, 테스트 VM에서만)
# innodb_doublewrite=0으로 설정 후 쓰기 중 강제 전원 차단
# → 데이터 파일 손상 여부 확인
```

### 실험 3: Kafka Sequential I/O 패턴 확인

```bash
# Kafka 메시지 생산 중 I/O 패턴 측정
$ iostat -xz 1 &

# 대량 메시지 생산
$ kafka-producer-perf-test.sh \
  --topic test-topic \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092

# iostat 결과 분석
# r/s 낮고 wMB/s 높음 → Sequential Write 확인
# await 낮음 → Sequential I/O 효율

# 세그먼트 파일 크기 성장 확인
$ watch -n 1 "ls -la /var/kafka-logs/test-topic-0/*.log"
# 파일 끝에 계속 추가됨 = Sequential Append
```

---

## 📊 성능/비용 비교

**HDD vs SSD Sequential/Random I/O 비교**:

```
                HDD (7200RPM)    SATA SSD     NVMe SSD
Sequential R    ~200 MB/s        ~550 MB/s    ~3500 MB/s
Sequential W    ~180 MB/s        ~500 MB/s    ~3000 MB/s
Random R (4K)   ~1 MB/s          ~100 MB/s    ~2000 MB/s
Random W (4K)   ~0.8 MB/s        ~90 MB/s     ~1500 MB/s
Seq/Rand 비율   200:1            5:1          2:1
```

**파티션 수에 따른 Kafka I/O 패턴**:

```
파티션 1개, 처리량 200MB/s:
  → 1개 파일에 Sequential Write
  → 디스크 효율 최대

파티션 10개, 처리량 200MB/s:
  → 10개 파일 라운드로빈 (각 20MB/s)
  → 파일 간 헤드 이동 (HDD) 또는 SSD Write Amp 증가
  → 실제 처리량 160MB/s로 저하 가능

파티션 100개:
  → 100개 파일 점프
  → HDD에서는 완전 Random I/O
  → NVMe SSD: 영향 제한적 (병렬 채널로 흡수)
```

---

## ⚖️ 트레이드오프

**Sequential I/O 설계의 제약**

Sequential 쓰기만 허용하면 업데이트(Update)나 랜덤 접근이 비효율적이다. Kafka가 Consumer 오프셋 추적을 별도 토픽(`__consumer_offsets`)으로 분리하고, 오래된 메시지를 삭제(Compaction)하는 것도 이 제약의 결과다. "수정 불가, 추가만 가능" 설계는 성능을 위한 의도적 선택이다.

**SSD 쓰기 증폭과 수명**

SSD의 쓰기 증폭(Write Amplification Factor, WAF)은 랜덤 쓰기가 많을수록 커진다. TLC NAND SSD의 경우 WAF가 최대 수십 배에 달할 수 있다. 엔터프라이즈 SSD가 더 많은 OP(Over-Provisioning)를 갖고, 랜덤 쓰기 워크로드에서도 WAF를 낮게 유지하는 이유가 여기에 있다.

---

## 📌 핵심 정리

```
HDD Sequential vs Random:
  HDD 성능 차이의 핵심 = 헤드 탐색(Seek Time) 유무
  Sequential: 헤드 이동 없음 → 200 MB/s
  Random 4KB: 헤드 이동 필수 → 1 MB/s (200배 차이!)

SSD Sequential vs Random:
  물리 탐색 없지만 여전히 Sequential이 유리
  이유: Write Amplification (랜덤 쓰기 → 블록 단위 삭제 증가)
  차이: 2~5배 (HDD의 100~200배보다 훨씬 작음)

MySQL Double Write Buffer:
  Dirty 페이지 → Sequential Write to DWB → Random Write to datafile
  Random I/O를 줄이고 파셜 페이지 쓰기 방지

Kafka Sequential Append:
  항상 파일 끝에 추가 = 완전 Sequential Write
  파티션 수가 많을수록 I/O 분산 → Random I/O 증가

fio로 측정:
  --rw=read/write   → Sequential
  --rw=randread/randwrite → Random
  --direct=1        → Page Cache 제거 (실제 디스크 성능)
  --bs=4k           → 블록 크기 (작을수록 IOPS, 클수록 처리량)
```

---

## 🤔 생각해볼 문제

**Q1.** MySQL InnoDB가 Double Write Buffer를 사용하면서도 `innodb_flush_method=O_DIRECT`를 함께 권장하는 이유는?

<details>
<summary>해설 보기</summary>

두 설정은 서로 다른 문제를 해결한다. Double Write Buffer는 파셜 페이지 쓰기 방지와 Sequential I/O 변환을 위한 것이다. `O_DIRECT`는 OS Page Cache와 InnoDB Buffer Pool 간의 이중 캐싱을 방지하기 위한 것이다. `O_DIRECT` 없이 DWB를 쓰면, DWB의 Sequential 쓰기 데이터가 Page Cache에도 캐싱되어 메모리가 낭비된다. 두 설정을 함께 사용하면 DWB로 쓰기 효율을 높이면서, Page Cache 우회로 메모리를 정확히 제어할 수 있다.

</details>

**Q2.** Kafka 브로커를 NVMe SSD 서버로 교체했을 때, 파티션 수 최적화 전략이 HDD 서버와 달라지는 이유는?

<details>
<summary>해설 보기</summary>

HDD에서는 파티션 수가 많아질수록 헤드 탐색이 증가해 Random I/O 패턴이 치명적이다. NVMe SSD는 랜덤과 순차의 성능 차이가 약 2배 수준이고, 내부 병렬 채널로 동시 I/O를 효율적으로 처리한다. 따라서 NVMe 서버에서는 파티션 수를 더 많이 늘려도 성능 저하가 제한적이다. 오히려 처리량과 병렬성을 위해 파티션을 더 많이 만드는 전략이 유효하다. 단, 파티션 수는 여전히 브로커 메모리와 리더 선출 비용을 고려해야 한다.

</details>

**Q3.** `fio --rw=randrw --rwmixread=70`으로 측정 시, 읽기 70% 쓰기 30% 혼합 워크로드가 읽기 100% 또는 쓰기 100%보다 성능이 낮은 이유는?

<details>
<summary>해설 보기</summary>

HDD에서는 읽기와 쓰기 헤드가 번갈아 서로 다른 위치로 이동하면서 탐색 거리가 증가한다. 순수 읽기/쓰기는 I/O 스케줄러가 같은 방향으로 정렬할 수 있지만, 혼합 I/O는 정렬 효율이 떨어진다. SSD에서도 읽기는 캐시에서, 쓰기는 빈 페이지에 기록하는 내부 로직이 혼합 패턴에서 최적화하기 어렵다. 또한 쓰기 중에 읽기 요청이 삽입되면 쓰기 버퍼 플러시가 지연되거나 읽기가 쓰기를 기다리는 대기가 발생한다.

</details>

---

**[⬅️ 이전: 디스크 I/O 스택](./02-disk-io-stack.md)** | **[홈으로 🏠](../README.md)** | **[다음: fsync와 내구성 ➡️](./04-fsync-durability.md)**
