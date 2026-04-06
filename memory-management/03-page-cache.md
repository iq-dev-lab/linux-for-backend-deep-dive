# 03. Page Cache — 디스크 I/O를 메모리에서 처리하는 원리

## 🎯 핵심 질문

- 같은 파일을 두 번 읽으면 왜 두 번째가 빠른가?
- `free -h`의 `buff/cache`는 실제로 무엇이고, 애플리케이션이 쓸 수 있는 메모리인가?
- MySQL `innodb_flush_method=O_DIRECT`가 왜 이중 캐싱을 방지하는가?
- Kafka가 디스크 기반임에도 빠른 진짜 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**`free -h`에서 `available`이 `free`보다 훨씬 크다.** `buff/cache`로 잡힌 메모리가 애플리케이션이 필요하면 즉시 반환되기 때문이다. 이를 모르면 "메모리가 부족하다"고 오판해 불필요하게 서버를 증설하는 일이 생긴다.

**MySQL 서버에서 `innodb_buffer_pool_size`와 OS Page Cache가 같은 데이터를 두 번 캐싱한다.** `O_DIRECT` 설정 없이 InnoDB가 파일을 읽으면 OS Page Cache에 한 번, InnoDB Buffer Pool에 한 번 — 같은 데이터가 메모리에 두 벌 존재한다. 물리 메모리 낭비다.

**Kafka는 디스크에 쓰는데 왜 빠른가?** Kafka는 Page Cache를 브로커의 메모리처럼 활용한다. Producer가 쓴 메시지는 Page Cache에 머물고, Consumer는 대부분 Page Cache에서 직접 읽는다. 메시지가 오래되지 않으면 디스크 I/O가 아예 발생하지 않는다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: buff/cache를 "낭비된 메모리"로 오해
$ free -h
              total   used   free   shared  buff/cache  available
Mem:           15Gi   3Gi   1Gi    256Mi       11Gi        11Gi

# "메모리 11GB가 캐시에 낭비되고 있어. free가 1GB밖에 없네"
# → 잘못된 판단. available 11GB = 실제로 쓸 수 있는 메모리
# → buff/cache는 애플리케이션 요청 시 즉시 반환되는 메모리

# 실수 2: 캐시 비우기로 "메모리 확보"
$ echo 3 > /proc/sys/vm/drop_caches
# → 일시적으로 buff/cache 감소
# → 다음 파일 접근 시 Page Cache 미스 → 성능 저하
# → 메모리가 실제로 부족한 문제는 해결 안 됨
# → 운영 환경에서 성능 저하 유발

# 실수 3: MySQL 이중 캐싱 방치
# innodb_buffer_pool_size = 8GB
# + OS Page Cache가 동일 파일 캐싱 = 추가 수 GB
# → 총 사용 메모리가 예상보다 훨씬 많음
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# free 올바른 해석
$ free -h
              total   used   free   buff/cache  available
Mem:           15Gi   3Gi   1Gi        11Gi        11Gi
#                                                   ↑
# available = 실제로 사용 가능한 메모리 (free + 회수 가능한 cache)

# Page Cache 현황 상세 확인
$ cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|Buffers|Cached|Dirty"
MemTotal:       16384000 kB
MemFree:         1048576 kB
MemAvailable:   12288000 kB   ← 실제 가용 메모리
Buffers:          102400 kB   ← 블록 장치 메타데이터 캐시
Cached:         11264000 kB   ← 파일 Page Cache
Dirty:            204800 kB   ← 수정됐지만 디스크에 아직 안 쓴 페이지

# MySQL O_DIRECT 설정으로 이중 캐싱 방지
# my.cnf:
# innodb_flush_method = O_DIRECT
# → InnoDB가 Page Cache 우회, Buffer Pool만 사용
```

---

## 🔬 내부 동작 원리

### Page Cache의 구조

```
Page Cache = 커널이 관리하는 파일 내용 캐시

파일 읽기 경로 (기본, Page Cache 사용):

  read("/data/mysql/ibdata1", buf, 16384)
         │
         ▼
  [VFS 레이어]
         │
         ▼
  [Page Cache 조회]
         │
    ┌────┴────┐
    │ 히트     │ 미스
    │         │
    ▼         ▼
  메모리에서  디스크에서 읽기
  직접 반환   → Page Cache에 저장
              → 이후 접근 시 캐시 히트
```

Page Cache는 물리 메모리에 파일의 4KB 페이지 단위로 캐싱된다. 파일의 어느 오프셋이 어느 물리 페이지에 있는지는 `radix tree`(최신 커널에서는 `xarray`)로 관리한다.

### `free -h` 항목 완전 해석

```
$ free -h
              total   used    free  shared  buff/cache  available
Mem:           15Gi   3Gi    1Gi    256Mi      11Gi        11Gi

total    = 물리 메모리 전체
used     = 프로세스가 실제 사용 중인 메모리
free     = 아무것도 없는 빈 메모리 (진짜 비어있음)
shared   = tmpfs 등 공유 메모리
buff/cache = Page Cache + 블록 버퍼
             → 파일 내용 + 파일시스템 메타데이터 캐시
             → 애플리케이션 요청 즉시 반환 가능
available  = free + 반환 가능한 buff/cache 일부
             → 실제로 새 프로세스가 쓸 수 있는 메모리
             ★ 이 값이 "사용 가능한 메모리"

Dirty Pages (중요):
  수정된 Page Cache 페이지 (아직 디스크에 안 쓴 것)
  → 즉시 반환 불가 (먼저 디스크에 써야 함)
  → Dirty 비율이 높으면 fsync 지연 가능
```

### MySQL과 Page Cache의 관계

```
innodb_flush_method = fsync (기본값):

  InnoDB가 파일 읽기:
    read() 호출 → OS Page Cache → 물리 메모리
  InnoDB Buffer Pool에도 동일 데이터 캐싱:
    → 같은 데이터가 메모리에 두 벌!

  물리 메모리 16GB, innodb_buffer_pool_size = 8GB:
    Buffer Pool: 8GB
    OS Page Cache (동일 파일): 추가 수 GB
    실제 사용: 12~14GB → 예상보다 많음

innodb_flush_method = O_DIRECT:

  InnoDB가 파일 읽기:
    pread() with O_DIRECT → Page Cache 완전 우회
    → 직접 InnoDB Buffer Pool에 적재
  Page Cache 사용 안 함:
    → 이중 캐싱 없음
    → Buffer Pool 8GB = 실제 캐시 8GB (정확한 예측)

  권장 설정:
    innodb_buffer_pool_size = 물리 메모리 * 0.7~0.8
    innodb_flush_method = O_DIRECT
```

### Kafka의 Page Cache 활용 전략

```
Kafka 아키텍처 (Page Cache 중심):

Producer → 메시지 전송
       │
       ▼
  Kafka Broker
  write() → OS Page Cache ← 물리 메모리에 적재
       │         │
       │         └── 비동기로 디스크에 쓰기 (Dirty 페이지)
       │
Consumer → 메시지 읽기 요청
       │
       ▼
  sendfile() 시스템 콜
       │
       ├── Page Cache HIT: 메모리에서 직접 Consumer로 전송
       │   (디스크 I/O 없음! Zero-copy)
       │
       └── Page Cache MISS: 디스크에서 읽기 → Cache → 전송

Kafka가 빠른 이유:
  1. Producer 쓰기: 디스크가 아닌 Page Cache에 즉시 반환
  2. Consumer 읽기: 최신 메시지는 Page Cache에서 직접
  3. sendfile(): Page Cache → NIC (사용자 공간 복사 없음)
  4. Sequential I/O: 디스크 접근 시도 최소화

Kafka 설정 최적화:
  → Heap 크기를 크게 잡지 말 것 (6GB 이하 권장)
  → 나머지 메모리를 Page Cache로 남겨둬야 성능 발휘
  → heap이 크면 GC가 Page Cache를 밀어냄
```

### Write-Back vs Write-Through

```
리눅스 기본 동작 (Write-Back):
  write() → Page Cache에 저장 (Dirty 표시)
  → 즉시 반환 (디스크 쓰기 안 함!)
  → 나중에 pdflush/writeback이 비동기로 디스크 기록

  장점: write() 응답이 빠름
  단점: 서버 크래시 시 Dirty 페이지 데이터 손실

fsync() 호출 시:
  → Dirty 페이지를 강제로 디스크에 플러시
  → 완료 후 반환 (느리지만 내구성 보장)

MySQL WAL(Write-Ahead Log):
  → 변경 사항을 먼저 redo log에 fsync()
  → Buffer Pool의 데이터 페이지는 나중에 쓰기
  → 성능과 내구성의 균형

Dirty 비율 제어 (vm.dirty_ratio):
  /proc/sys/vm/dirty_ratio = 20     ← 물리 메모리의 20% 초과 시 강제 flush
  /proc/sys/vm/dirty_background_ratio = 10  ← 10% 초과 시 백그라운드 flush 시작
```

---

## 💻 실전 실험

### 실험 1: Page Cache 효과 직접 측정

```bash
# 테스트 파일 생성 (1GB)
$ dd if=/dev/urandom of=/tmp/test_file bs=1M count=1024

# 1차 읽기 (Page Cache 미스 - 디스크에서)
$ echo 3 > /proc/sys/vm/drop_caches  # Page Cache 비우기
$ time dd if=/tmp/test_file of=/dev/null bs=1M
# 결과 예시: 1073741824 bytes transferred in 2.5 secs (428 MB/s) ← 디스크 속도

# 2차 읽기 (Page Cache 히트 - 메모리에서)
$ time dd if=/tmp/test_file of=/dev/null bs=1M
# 결과 예시: 1073741824 bytes transferred in 0.3 secs (3.5 GB/s) ← 메모리 속도
# 약 8~10배 빠름!

# Page Cache에 파일이 올라온 것 확인
$ cat /proc/meminfo | grep Cached
Cached: 1048576 kB   ← 1GB 증가 (test_file이 캐싱됨)
```

### 실험 2: MySQL O_DIRECT vs fsync 이중 캐싱 확인

```bash
# Docker로 MySQL 두 인스턴스 실행 (설정만 다르게)
$ docker run -d --name mysql-fsync \
  -e MYSQL_ROOT_PASSWORD=test \
  -e MYSQL_DATABASE=testdb \
  mysql:8 --innodb-flush-method=fsync

$ docker run -d --name mysql-odirect \
  -e MYSQL_ROOT_PASSWORD=test \
  -e MYSQL_DATABASE=testdb \
  mysql:8 --innodb-flush-method=O_DIRECT

# 데이터 로드 후 메모리 사용량 비교
$ for name in mysql-fsync mysql-odirect; do
    PID=$(docker inspect --format='{{.State.Pid}}' $name)
    RSS=$(awk '/VmRSS/{print $2}' /proc/$PID/status)
    echo "$name RSS: ${RSS}kB"
  done

# mysql-fsync: 더 높은 RSS (Buffer Pool + Page Cache 이중 캐싱)
# mysql-odirect: 더 낮은 RSS (Buffer Pool만)
```

### 실험 3: Dirty 페이지 모니터링

```bash
# Dirty 페이지 비율 실시간 확인
$ watch -n 1 'cat /proc/meminfo | grep -E "Dirty|Writeback"'
Dirty:     512000 kB   ← 아직 디스크에 안 쓴 페이지
Writeback:   4096 kB   ← 현재 디스크에 쓰는 중

# 대량 쓰기 후 Dirty 증가 관찰
$ dd if=/dev/urandom of=/tmp/dirty_test bs=1M count=2048 &
$ watch -n 0.2 'cat /proc/meminfo | grep Dirty'
# Dirty가 증가하다가 vm.dirty_ratio 한도에 가까워지면
# 쓰기 프로세스가 블로킹되어 fsync 대기 발생
```

---

## 📊 성능/비용 비교

| 시나리오 | Page Cache 히트 | Page Cache 미스 |
|---------|---------------|----------------|
| 파일 읽기 속도 | ~GB/s (메모리 속도) | ~100~500 MB/s (SSD) |
| 응답 시간 | ~수십 μs | ~수 ms |
| CPU 사용 | 낮음 | 높음 (I/O 대기) |

**MySQL flush method별 메모리 사용 비교**:

```
물리 메모리 16GB, innodb_buffer_pool_size = 8GB

innodb_flush_method = fsync:
  Buffer Pool:     8GB
  OS Page Cache:   ~4GB (동일 datafile 캐싱)
  기타:            ~2GB
  합계:            ~14GB 사용

innodb_flush_method = O_DIRECT:
  Buffer Pool:     8GB
  OS Page Cache:   ~0 (우회)
  기타:            ~2GB
  합계:            ~10GB 사용
  → 4GB 절감!
```

**Kafka Page Cache 활용률과 성능**:

```
Consumer lag = 0 (최신 메시지 소비):
  Page Cache 히트율: ~100%
  처리량: 메모리 I/O 속도 (수 GB/s)

Consumer lag = 10분 (10분치 메시지 적체):
  Page Cache 히트율: Page Cache 크기에 따라 다름
  메시지 크기 * 10분 < Page Cache → 여전히 캐시 히트
  메시지 크기 * 10분 > Page Cache → 디스크 I/O 발생
```

---

## ⚖️ 트레이드오프

**Page Cache의 딜레마**

Page Cache가 클수록 파일 I/O 성능이 좋아지지만, 애플리케이션이 사용할 메모리가 줄어든다. 반대로 애플리케이션에 메모리를 많이 주면 Page Cache 효율이 떨어진다. Redis처럼 데이터를 직접 메모리에 관리하는 서비스는 Page Cache가 필요 없고, Kafka처럼 Page Cache를 전략적으로 활용하는 서비스는 Heap을 작게 유지해야 한다.

**Dirty 비율과 쓰기 성능**

`vm.dirty_ratio`를 높이면 더 많은 데이터를 메모리에 모아서 배치로 디스크에 쓸 수 있어 I/O 효율이 좋다. 하지만 크래시 시 손실되는 데이터가 많아진다. DB 서버는 `fsync()`로 이 문제를 직접 해결하므로 OS의 Dirty 비율 설정을 낮춰도 무방하다.

---

## 📌 핵심 정리

```
Page Cache:
  파일 내용을 물리 메모리에 캐싱
  두 번째 읽기는 메모리 속도 (~GB/s)
  쓰기는 Page Cache에만 → 비동기 디스크 기록 (Write-Back)

free -h 해석:
  available = 실제 가용 메모리 (★ 이 값이 중요)
  buff/cache = 즉시 회수 가능한 캐시 (부족 아님)
  Dirty = 아직 디스크에 안 쓴 페이지 (모니터링 필요)

MySQL Page Cache 연계:
  O_DIRECT: Buffer Pool만 사용, 이중 캐싱 방지
  fsync: Page Cache + Buffer Pool 이중 캐싱 (메모리 낭비)

Kafka Page Cache 연계:
  Heap 최소화 → Page Cache 최대화 → Consumer 읽기 캐시 히트
  sendfile()로 Page Cache → NIC 제로 카피

핵심 진단 명령어:
  /proc/meminfo의 Cached, Dirty, MemAvailable
  vmstat -s | grep -i cache
  fincore (파일이 Page Cache에 있는지 확인)
  iostat -x: 디스크 I/O 발생 여부 (Page Cache 미스 간접 지표)
```

---

## 🤔 생각해볼 문제

**Q1.** Kafka 브로커의 JVM Heap을 32GB로 설정했다. 서버 물리 메모리는 64GB다. 이 설정의 문제점은?

<details>
<summary>해설 보기</summary>

Page Cache가 32GB로 줄어든다. Kafka의 성능은 Page Cache 크기에 직접 비례하므로 성능이 크게 저하된다. 또한 대용량 Heap에서 GC(특히 Full GC) 시간이 길어져 Consumer에게 레이턴시 스파이크가 발생한다. Confluent 권장 설정은 Heap을 6GB 이하로 유지하는 것이다. Kafka 브로커는 JVM 기반이지만 실제 데이터 처리는 OS Page Cache에 위임한다.

</details>

**Q2.** `echo 3 > /proc/sys/vm/drop_caches`를 운영 중 MySQL 서버에서 실행하면 어떤 일이 일어나는가?

<details>
<summary>해설 보기</summary>

Page Cache가 비워지면서 MySQL의 `innodb_flush_method=fsync` 설정 시 ibdata, redo log 등의 파일이 캐시에서 제거된다. 이후 MySQL 쿼리들이 파일 접근 시 Page Cache 미스가 발생해 모두 디스크 I/O를 수행한다. 순간적으로 디스크 I/O가 폭증하고 쿼리 응답 시간이 수십~수백 배 증가할 수 있다. 운영 환경에서 절대 실행하면 안 된다. `O_DIRECT` 설정 시에는 MySQL이 Page Cache를 쓰지 않으므로 영향이 적다.

</details>

**Q3.** Redis에서 AOF를 `appendfsync everysec`으로 설정했다. 매 초 `fsync()`가 호출될 때 Page Cache의 Dirty 페이지는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`fsync()`는 해당 파일 디스크립터에 연결된 Dirty 페이지만 디스크에 강제 기록한다. AOF 파일의 Dirty 페이지가 초당 한 번씩 디스크에 반영되므로 최대 1초치 데이터만 손실될 수 있다. `fsync()` 호출은 동기적으로 완료될 때까지 블로킹되므로, 디스크가 느린 환경에서는 매 초 수십 ms의 지연이 Redis 이벤트 루프에 영향을 줄 수 있다. 이를 완화하기 위해 AOF `fsync()`를 별도 스레드에서 처리한다.

</details>

---

**[⬅️ 이전: Page Fault](./02-page-fault.md)** | **[홈으로 🏠](../README.md)** | **[다음: mmap과 Direct I/O ➡️](./04-mmap-direct-io.md)**
