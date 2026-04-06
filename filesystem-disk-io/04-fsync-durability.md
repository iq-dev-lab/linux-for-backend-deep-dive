# 04. fsync와 내구성 — 커널 버퍼 플러시의 시점

## 🎯 핵심 질문

- `write()` 후 데이터는 어디에 있고, 서버가 재시작되면 어떻게 되는가?
- `fsync()`와 `fdatasync()`는 무엇이 다른가?
- MySQL `innodb_flush_method`의 각 옵션은 커널에서 실제로 어떻게 동작하는가?
- `fsync()` 없이 운영 중인 서비스에서 데이터가 유실되는 정확한 시나리오는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**MySQL 서버가 갑자기 꺼졌다가 재시작됐는데 데이터가 없다.** `innodb_flush_log_at_trx_commit=0` 또는 `2`로 설정된 상태에서 장애가 발생하면, Page Cache에만 있던 redo log 데이터가 사라진다. 이 설정 하나가 "커밋된 트랜잭션도 유실될 수 있음"을 의미한다.

**Redis AOF `appendfsync=no`로 설정한 서버가 재시작 후 최대 30초치 데이터를 잃었다.** OS가 결정하는 Write-Back 주기(기본 30초)가 지나기 전에 장애가 발생하면, Dirty 페이지의 모든 데이터가 사라진다. `appendfsync=everysec`이 "최대 1초 손실"을 보장하는 이유가 여기에 있다.

**`fsync()`를 너무 자주 호출해서 MySQL 처리량이 절반으로 떨어졌다.** `innodb_flush_log_at_trx_commit=1`은 모든 커밋마다 `fsync()`를 호출한다. SSD에서도 `fsync()`는 수 ms가 소요되므로, 초당 수백 트랜잭션에서 병목이 된다. 내구성과 처리량 사이의 트레이드오프를 정확히 이해해야 한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: write() 후 데이터가 즉시 디스크에 저장됐다고 가정
$ echo "important data" > /data/critical.txt
# → Page Cache에만 저장! 디스크 동기화는 나중에 비동기로
# → 이 시점에 전원이 꺼지면 데이터 유실

# 올바른 방법:
$ echo "important data" > /data/critical.txt && sync
# 또는 코드에서: fsync(fd) 호출

# 실수 2: MySQL 내구성 설정을 성능을 위해 낮추고 위험을 인지 못 함
# my.cnf:
# innodb_flush_log_at_trx_commit = 0  ← "성능이 빠르네"
# → 장애 시 마지막 1초 이상의 커밋된 트랜잭션 유실 가능

# 실수 3: Redis AOF rewrite 중 데이터 유실 가능성 무시
# appendfsync = no
# → OS Write-Back 주기(30초) 동안 Dirty 페이지 누적
# → 장애 시 최대 30초치 손실
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 현재 Dirty 페이지 현황
$ cat /proc/meminfo | grep Dirty
Dirty:    204800 kB   ← 아직 디스크에 안 쓴 데이터 200MB
Writeback:  4096 kB   ← 현재 쓰는 중

# Write-Back 설정 확인
$ sysctl vm.dirty_expire_centisecs   # Dirty 페이지 최대 수명 (기본 3000 = 30초)
$ sysctl vm.dirty_writeback_centisecs # Write-Back 주기 (기본 500 = 5초)

# 서비스 특성에 맞는 MySQL 설정
# OLTP (금융, 결제): 내구성 최우선
# my.cnf:
# innodb_flush_log_at_trx_commit = 1  ← 모든 커밋에 fsync
# innodb_flush_method = O_DIRECT

# 분석/배치: 처리량 우선, 약간의 손실 허용
# innodb_flush_log_at_trx_commit = 2  ← 초당 1회 fsync
# sync_binlog = 100  ← 100개 트랜잭션마다 binlog sync
```

---

## 🔬 내부 동작 원리

### write() 후 데이터의 위치

```
write(fd, "Hello", 5) 호출 후:

[애플리케이션 버퍼]
  "Hello"
       │ write() 시스템 콜
       ▼
[커널 Page Cache]
  해당 파일 페이지에 "Hello" 기록
  Dirty 플래그 설정
  → 여기까지: 즉시 반환 (빠름)
  → 전원 장애 시 여기 있는 데이터 유실

       │ (나중에, 비동기)
       ▼ pdflush/writeback 커널 스레드
[디스크]
  실제 영구 저장
  → 이 시점부터: 장애에도 생존

Write-Back 트리거:
  1. vm.dirty_expire_centisecs (기본 30초) 이상 된 Dirty 페이지
  2. vm.dirty_ratio (기본 20%) 초과 시 강제 플러시
  3. fsync() 호출
  4. sync 명령어
```

### fsync() vs fdatasync() 동작

```
fsync(fd):
  1. 해당 fd에 연결된 모든 Dirty 페이지 디스크에 기록
  2. 파일 메타데이터(inode) 업데이트도 디스크에 기록
     (크기, 수정 시간, atime 등)
  3. 하드웨어 쓰기 캐시 플러시 신호 전송
  4. 물리 디스크 확인 후 반환

fdatasync(fd):
  1. 해당 fd의 데이터 Dirty 페이지 디스크에 기록
  2. 메타데이터는 건너뜀 (크기 변경된 경우만 업데이트)
  3. 메타데이터 I/O 없음 → fsync보다 빠름

비용 차이:
  SSD에서 fsync():    ~0.5~2ms
  SSD에서 fdatasync(): ~0.3~1ms (메타데이터 I/O 생략)
  HDD에서 fsync():    ~5~15ms (헤드 이동 포함)

실용적 선택:
  데이터 무결성이 중요한 DB → fsync() (메타데이터도 중요)
  로그 파일 순차 쓰기 → fdatasync() (크기만 변하므로 충분)
  Redis AOF → fdatasync() 사용
```

### MySQL innodb_flush_method 옵션별 커널 동작

```
옵션 1: fsync (기본값)
  redo log 쓰기:  write() → Page Cache → fsync() → 디스크
  data 쓰기:      write() → Page Cache → fsync() → 디스크
  특징: Page Cache 사용 (이중 캐싱 발생)

옵션 2: O_DIRECT
  redo log 쓰기:  pwrite(O_DIRECT) → 디스크 (Page Cache 우회)
  data 쓰기:      pwrite(O_DIRECT) → 디스크 (Page Cache 우회)
  fsync() 호출:   O_DIRECT라도 하드웨어 캐시 플러시 위해 fsync 필요
  특징: 이중 캐싱 없음, Buffer Pool이 유일한 캐시

옵션 3: O_DSYNC
  redo log 쓰기:  pwrite(O_DSYNC) → 즉시 디스크 동기화
                  (fdatasync와 동일 효과, 쓸 때마다 동기화)
  data 쓰기:      write() → Page Cache → fsync()
  특징: redo log는 즉시 내구성, data는 별도 fsync

옵션 4: O_DIRECT_NO_FSYNC
  O_DIRECT + fsync() 생략
  배터리 지원 RAID 컨트롤러가 내구성 보장 시 사용
  위험: 컨트롤러 장애 시 데이터 손실

시스템 콜 비교:
  fsync:          write() + fsync()
  O_DIRECT:       pwrite(O_DIRECT) + fsync()
  O_DSYNC:        pwrite(O_DSYNC) [= write + fdatasync 효과]
```

### innodb_flush_log_at_trx_commit 설정

```
값 = 1 (기본, 최고 내구성):
  커밋 시 → redo log 버퍼 write() → 즉시 fsync()
  → 커밋된 트랜잭션은 장애 시에도 100% 보장
  → 처리량: 낮음 (커밋마다 fsync)

값 = 2 (중간):
  커밋 시 → redo log 버퍼 write() → Page Cache (fsync 없음)
  백그라운드 스레드 → 초당 1회 fsync()
  → 최대 1초치 커밋된 트랜잭션 손실 가능
  → 처리량: 높음 (초당 1회 fsync)

값 = 0 (최고 성능, 최악 내구성):
  커밋 시 → redo log 버퍼에만 (write도 안 함!)
  백그라운드 스레드 → 초당 1회 write + fsync
  → MySQL 크래시 시 최대 1초 손실
  → OS 크래시 시 더 많은 손실 가능
  → 처리량: 최고 (fsync 거의 없음)

실무 가이드:
  금융/결제/의료: flush=1 (손실 불허)
  일반 서비스:   flush=2 (1초 손실 허용, 처리량 중요)
  분석/로그:     flush=0 (성능 최우선, 손실 허용)
```

### 데이터 유실 시나리오와 예방

```
시나리오 1: write() 직후 전원 장애
  write(fd, data, size) → Page Cache에 저장
  [전원 꺼짐]
  → Page Cache 날아감 → 데이터 유실
  예방: fsync() 호출 후 반환 (파일 저장 도구, DB WAL)

시나리오 2: 파일 생성 + 디렉토리 업데이트 미스
  create("/data/new_file.txt")   → 파일 내용 fsync()
  [장애]
  → 파일 내용은 디스크에 있지만 디렉토리 엔트리 없음
  → 파일을 찾을 수 없음! (orphan inode)
  예방: 파일 내용 fsync() + 부모 디렉토리 fsync()

시나리오 3: rename() 기반 원자적 업데이트
  올바른 패턴:
    write("config.tmp", new_data)  → 임시 파일
    fsync("config.tmp")            → 임시 파일 내구성
    rename("config.tmp", "config") → 원자적 대체
    fsync(parent_directory_fd)     → 디렉토리 업데이트 내구성
  
  이 패턴이 없으면:
    "config" 파일이 쓰기 중 장애 발생 시 파일 손상
    tmpfile + rename + dir fsync = 원자적 + 내구적 업데이트
```

---

## 💻 실전 실험

### 실험 1: fsync() 비용 측정

```c
/* fsync_cost.c */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <time.h>
#include <string.h>

long now_ns() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1e9 + ts.tv_nsec;
}

int main() {
    int fd = open("/tmp/fsync_test", O_WRONLY|O_CREAT|O_TRUNC, 0644);
    char buf[4096];
    memset(buf, 'A', sizeof(buf));
    int N = 1000;

    /* write()만 N번 */
    long start = now_ns();
    for (int i = 0; i < N; i++) write(fd, buf, 4096);
    printf("write() only   %d회: %ldms\n", N, (now_ns()-start)/1000000);

    /* write() + fsync() N번 */
    lseek(fd, 0, SEEK_SET);
    start = now_ns();
    for (int i = 0; i < N; i++) {
        write(fd, buf, 4096);
        fsync(fd);
    }
    printf("write()+fsync() %d회: %ldms\n", N, (now_ns()-start)/1000000);

    close(fd);
    unlink("/tmp/fsync_test");
    return 0;
}
```

```bash
$ gcc fsync_cost.c -o fsync_cost && ./fsync_cost
write() only    1000회:   12ms   ← Page Cache, 즉시 반환
write()+fsync() 1000회: 1850ms   ← SSD 기준 fsync 1.85ms * 1000
# → fsync가 처리량에 미치는 영향 직접 확인
```

### 실험 2: Dirty 페이지 Write-Back 시뮬레이션

```bash
# Write-Back 설정 확인 및 조정
$ sysctl vm.dirty_expire_centisecs vm.dirty_writeback_centisecs
vm.dirty_expire_centisecs = 3000    # 30초 후 Dirty 페이지 flush
vm.dirty_writeback_centisecs = 500  # 5초마다 writeback 스레드 깨어남

# 대량 쓰기 후 Dirty 페이지 누적 관찰
$ dd if=/dev/zero of=/tmp/dirty_test bs=1M count=500 &
$ watch -n 0.5 'cat /proc/meminfo | grep -E "Dirty|Writeback"'
# Dirty가 증가하다가 vm.dirty_ratio 한도 도달 시 강제 flush

# 즉시 flush 강제 (운영 환경 주의)
$ sync  # 또는
$ echo 1 > /proc/sys/vm/dirty_expire_centisecs && sync
```

### 실험 3: MySQL 내구성 설정별 처리량 측정

```bash
# sysbench로 MySQL OLTP 처리량 측정

# 설정 1: 최고 내구성 (flush=1)
$ mysql -e "SET GLOBAL innodb_flush_log_at_trx_commit=1;"
$ sysbench oltp_write_only --mysql-host=127.0.0.1 \
  --tables=4 --table-size=100000 --threads=8 --time=60 run
# TPS 예시: 1,200 TPS

# 설정 2: 중간 (flush=2)
$ mysql -e "SET GLOBAL innodb_flush_log_at_trx_commit=2;"
$ sysbench oltp_write_only ... run
# TPS 예시: 4,800 TPS (4배 향상)

# 설정 3: 최고 성능 (flush=0)
$ mysql -e "SET GLOBAL innodb_flush_log_at_trx_commit=0;"
$ sysbench oltp_write_only ... run
# TPS 예시: 6,500 TPS (5.4배 향상)

# iostat로 각 설정에서 fsync 빈도 비교
$ iostat -xz 1  # w/s: 설정 1이 가장 많음 (매 커밋마다 쓰기)
```

---

## 📊 성능/비용 비교

**fsync() 비용 비교 (디스크 유형별)**:

```
SSD (SATA):    fsync() ~1~3ms
SSD (NVMe):    fsync() ~0.1~0.5ms
HDD (7200RPM): fsync() ~5~15ms
배터리 RAID:   fsync() ~0.01ms (쓰기 캐시에 도달 즉시)

초당 처리 가능한 fsync() 수:
  NVMe:    2,000~10,000 / 초
  SSD:     333~1,000 / 초
  HDD:     66~200 / 초
```

**MySQL innodb_flush_log_at_trx_commit 설정별 트레이드오프**:

```
설정   내구성              처리량   손실 위험
  1    100% 보장           낮음     없음
  2    ~1초 손실 가능      높음     작음 (OS/하드웨어 장애 시만)
  0    1초+ 손실 가능      최고     있음 (MySQL 크래시 시도)

Redis appendfsync:
  always:   커밋마다 fsync  → 내구성 최고, 성능 낮음
  everysec: 초당 1회 fsync  → 1초 손실, 성능 좋음 (권장)
  no:       OS 결정         → 손실 크게 가능, 성능 최고
```

---

## ⚖️ 트레이드오프

**내구성 vs 처리량의 근본적 긴장**

`fsync()`는 하드웨어 쓰기 캐시까지 플러시하는 가장 확실한 내구성 보장이지만, 그만큼 비싸다. 내구성 요구 사항이 낮은 서비스에서 `fsync()`를 남발하면 처리량이 불필요하게 줄어든다. 반대로 금융 시스템에서 `fsync()`를 생략하면 "커밋 완료" 응답을 받았음에도 데이터가 사라지는 치명적 상황이 발생한다. 서비스의 데이터 손실 허용 수준(RPO)을 먼저 정의하고 설정을 결정해야 한다.

**배터리 지원 RAID와 내구성의 관점**

엔터프라이즈 RAID 컨트롤러는 배터리(BBU)로 쓰기 캐시를 보호한다. 이 경우 `fsync()`가 쓰기 캐시에 도달하는 즉시 완료를 반환하고, 전원 장애 시 배터리로 디스크에 최종 기록한다. 이 환경에서 `O_DIRECT_NO_FSYNC`나 `innodb_flush_log_at_trx_commit=2`가 안전하게 쓰일 수 있는 이유다.

---

## 📌 핵심 정리

```
write() 후 데이터 위치:
  → Page Cache (Dirty 페이지)
  → 전원 장애 시 유실 가능!
  → fsync() 호출 후에야 디스크에 영구 저장

fsync() vs fdatasync():
  fsync():      데이터 + 메타데이터 모두 디스크
  fdatasync():  데이터만 디스크 (빠름, 순차 파일에 적합)

MySQL 설정:
  innodb_flush_log_at_trx_commit:
    1: 커밋마다 fsync (내구성 최고, 처리량 낮음)
    2: 초당 1회 fsync (1초 손실 허용, 처리량 높음)
    0: 비동기 (손실 가능, 처리량 최고)
  innodb_flush_method:
    O_DIRECT: Page Cache 우회 (이중 캐싱 방지)
    fsync:    Page Cache 경유 (기본값)

원자적 파일 업데이트 패턴:
  write(tmp) → fsync(tmp) → rename(tmp, target) → fsync(dir)

핵심 진단:
  /proc/meminfo Dirty     → 유실 위험 데이터 크기
  vm.dirty_expire_centisecs → Write-Back 최대 지연
  strace -e trace=fsync   → 애플리케이션의 fsync 패턴
```

---

## 🤔 생각해볼 문제

**Q1.** Redis AOF `appendfsync=everysec`이 "최대 1초 손실"을 보장한다. 실제로 1초보다 더 많은 데이터가 손실될 수 있는 경우는 언제인가?

<details>
<summary>해설 보기</summary>

`everysec`는 백그라운드 스레드가 초당 1번 `fdatasync()`를 호출한다. 만약 이전 `fdatasync()`가 완료되지 않았다면(디스크 포화 등), 다음 호출을 건너뛸 수 있다. 극단적인 경우 2~3초치 데이터가 손실될 수 있다. 또한 AOF Rewrite 중에는 새 AOF 파일에 쓰다가 장애 발생 시 Rewrite 이전 데이터가 기준이 되어 더 많이 손실될 수 있다. `no-appendfsync-on-rewrite yes`(기본값)로 설정하면 AOF Rewrite 중에는 `fsync()`를 하지 않아 위험이 더 커진다.

</details>

**Q2.** `rename("config.tmp", "config")` 호출 자체는 원자적이다. 그런데 왜 디렉토리 fd에 대해 추가로 `fsync()`를 호출해야 하는가?

<details>
<summary>해설 보기</summary>

`rename()`은 디렉토리 엔트리를 업데이트하는 작업이다. 이 업데이트가 Page Cache에만 반영되고 디스크에 기록되기 전에 장애가 발생하면, 재시작 후 파일 시스템에서 "config" 파일이 사라진다(old 항목 삭제 + new 항목 추가가 모두 손실). 디렉토리 fd로 `fsync()`를 호출하면 디렉토리 블록의 변경사항이 디스크에 영구 반영된다. ext4의 `data=ordered` 마운트 옵션은 데이터가 메타데이터보다 먼저 디스크에 기록됨을 보장하지만, 메타데이터 자체의 내구성은 별도 `fsync()`가 필요하다.

</details>

**Q3.** NVMe SSD 서버에서 MySQL `innodb_flush_log_at_trx_commit=1`로 운영 중이다. TPS가 낮아 `2`로 변경하고 싶다. 정확히 어떤 위험을 감수하는 것인가?

<details>
<summary>해설 보기</summary>

설정 `2`로 변경하면, 커밋 후 즉시 `fsync()`하지 않고 초당 1회만 한다. 이 1초 안에 장애가 발생하면 그 사이에 커밋 완료 응답을 받은 트랜잭션들이 유실된다. "커밋 완료"를 클라이언트에게 보냈는데 실제로는 사라지는 것이다. OS 커널 크래시나 하드웨어 장애에서는 최대 1초치 손실이 발생한다. 단, MySQL 프로세스만 비정상 종료된 경우(`kill -9`)에는 `2`도 안전하다 — OS의 Page Cache는 남아있어 MySQL 재시작 후 회복된다. 실질적 위험은 OS/하드웨어 장애 시나리오에서만 발생한다.

</details>

---

**[⬅️ 이전: Sequential vs Random I/O](./03-sequential-vs-random-io.md)** | **[홈으로 🏠](../README.md)** | **[다음: inode와 파일 시스템 구조 ➡️](./05-inode-filesystem.md)**
