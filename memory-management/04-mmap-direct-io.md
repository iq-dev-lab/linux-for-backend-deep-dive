# 04. mmap과 Direct I/O — Page Cache 활용과 우회

## 🎯 핵심 질문

- `mmap()`으로 파일을 매핑하면 `read()`와 무엇이 다른가?
- `O_DIRECT`는 Page Cache를 어떻게 우회하고, 왜 이게 필요한가?
- MySQL은 언제 `mmap()`을 쓰고 언제 `O_DIRECT`를 쓰는가?
- RocksDB와 같은 스토리지 엔진이 Direct I/O를 선호하는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**MySQL 서버의 메모리 사용량이 `innodb_buffer_pool_size` 설정을 훨씬 초과한다.** `innodb_flush_method=fsync`(기본값)일 때 InnoDB가 파일을 읽으면 OS Page Cache에 자동으로 캐싱된다. 이 Page Cache를 OS가 회수하기 전까지 Buffer Pool + Page Cache 이중으로 메모리를 사용한다. `O_DIRECT`로 전환하면 해결된다.

**Elasticsearch가 자체 캐시가 있는데도 OS의 Page Cache를 중요하게 생각하는 이유는?** Lucene 인덱스 파일을 `mmap()`으로 읽기 때문이다. `mmap()`은 Page Cache와 직접 연결되어 있어 OS가 자동으로 관리해준다. Elasticsearch JVM Heap을 물리 메모리의 50% 이하로 유지하라는 권장 사항이 나머지 50%를 Page Cache(=Lucene 인덱스 캐시)로 남겨두기 위한 것이다.

**RocksDB(Kafka, TiKV 등에 사용)가 Direct I/O를 제공하는 이유는?** RocksDB가 자체 Block Cache를 관리한다. OS Page Cache까지 겹치면 이중 캐싱이 발생하고, Block Cache의 Eviction 정책이 무의미해진다. Direct I/O로 Page Cache를 우회하면 RocksDB가 정확하게 메모리를 제어할 수 있다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```java
// 실수 1: 대용량 파일을 read()로 반복 읽기
// FileInputStream으로 100MB 파일을 반복 처리
FileInputStream fis = new FileInputStream("/data/large_file.bin");
byte[] buf = new byte[4096];
while (fis.read(buf) != -1) {
    process(buf);
}
// → 매번 시스템 콜 + 커널→유저 공간 복사 발생
// → mmap()이면 복사 없이 직접 접근 가능

// 실수 2: Direct I/O의 정렬 요구사항을 모름
int fd = open("file", O_RDONLY | O_DIRECT);
char buf[1000];  // 4096의 배수가 아님!
read(fd, buf, 1000);  // EINVAL 오류 발생!
// O_DIRECT는 버퍼 크기와 오프셋이 섹터 크기(보통 512B 또는 4096B)의 배수여야 함
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```c
/* mmap으로 파일 읽기 (복사 없음) */
#include <sys/mman.h>
#include <fcntl.h>
#include <sys/stat.h>

int fd = open("/data/large_file.bin", O_RDONLY);
struct stat sb;
fstat(fd, &sb);

// 파일을 가상 주소 공간에 직접 매핑
void *addr = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
// → Page Cache와 직접 연결
// → 접근 시 Page Fault → Page Cache에서 물리 페이지 매핑
// → 커널→유저 복사 없음

// 순차 접근 힌트 (Read-Ahead 최적화)
madvise(addr, sb.st_size, MADV_SEQUENTIAL);

// 접근 (Page Fault로 Page Cache에서 자동 로딩)
process_data(addr, sb.st_size);

munmap(addr, sb.st_size);
```

```bash
# MySQL O_DIRECT 설정 및 효과 확인
# my.cnf:
# innodb_flush_method = O_DIRECT

# 설정 적용 후 Page Cache 사용 여부 확인
$ iostat -x 1  # r/s가 Buffer Pool 미스만큼 발생하는지 확인
$ vmstat -s | grep "page cache"  # Page Cache 크기 변화 없어야 함
```

---

## 🔬 내부 동작 원리

### read() vs mmap() 비교

```
read() 시스템 콜 경로:

애플리케이션 (유저 공간)
  │  read(fd, user_buf, size)
  │
  ▼ [시스템 콜 경계]
커널 공간
  │  1. Page Cache 조회
  │     히트: Page Cache → user_buf 복사 (memcpy)
  │     미스: 디스크 → Page Cache → user_buf 복사
  │
  ▼ [시스템 콜 반환]
애플리케이션
  → user_buf에 데이터 존재

총 복사 횟수: 1회 (Page Cache → user_buf)
시스템 콜 오버헤드: 매번 발생

mmap() 경로:

애플리케이션 (유저 공간)
  │  addr = mmap(fd, ...)
  │  → 가상 주소 공간에 파일 매핑 (Page Table 설정만)
  │  → 실제 물리 메모리 할당 안 됨
  │
  addr[offset] 접근:
  → Page Fault (처음 접근 시)
  → 커널: Page Cache 확인
     히트: Page Cache 물리 페이지를 가상 주소에 직접 매핑
     미스: 디스크 → Page Cache → 가상 주소 매핑
  → 이후 접근: 시스템 콜 없이 직접 메모리 접근

총 복사 횟수: 0회 (Page Cache 페이지를 직접 매핑)
시스템 콜 오버헤드: 최초 1회 (mmap) + Page Fault만
```

### mmap 내부 동작

```
mmap() 호출 시:

1. 커널: VMA(Virtual Memory Area) 생성
   → 가상 주소 범위 예약
   → file + offset 정보 기록
   → 물리 메모리 할당 안 함

2. 첫 접근 시 Page Fault:
   do_fault() 호출
   → 파일의 해당 페이지가 Page Cache에 있는가?
     있음: filemap_map_pages() → PTE에 Page Cache 물리 주소 직접 등록
     없음: 디스크에서 읽기 → Page Cache 적재 → PTE 등록

3. 이후 접근:
   → TLB 히트 or PTE 조회 → 물리 주소 직접 접근
   → 시스템 콜 없음, 복사 없음

madvise()로 접근 패턴 힌트:
  MADV_SEQUENTIAL: Read-Ahead 활성화 (순차 읽기)
  MADV_RANDOM:     Read-Ahead 비활성화 (랜덤 접근)
  MADV_WILLNEED:   미리 Page Cache에 로딩 요청
  MADV_DONTNEED:   Page Cache에서 제거 허용
```

### O_DIRECT — Page Cache 완전 우회

```
일반 I/O (Page Cache 경유):

  디스크 → DMA → Page Cache (커널 메모리)
                       │
                       └── memcpy → 애플리케이션 버퍼

Direct I/O (O_DIRECT):

  디스크 → DMA → 애플리케이션 버퍼 (직접!)
             ↑
         DMA가 바로 유저 공간 버퍼에 쓰기

제약 조건 (O_DIRECT 사용 시 반드시 지켜야 함):
  1. 버퍼 주소: 섹터 크기(512B 또는 4096B)의 배수 정렬
  2. 오프셋: 섹터 크기의 배수
  3. 읽기/쓰기 크기: 섹터 크기의 배수
  위반 시: EINVAL 오류

장점:
  Page Cache 메모리 사용 없음
  애플리케이션이 직접 캐싱 전략 제어
  이중 캐싱 방지

단점:
  Read-Ahead 없음 (순차 읽기 최적화 사라짐)
  Write Combining 없음 (작은 쓰기가 개별 디스크 I/O)
  복잡한 버퍼 정렬 요구사항
```

### MySQL의 flush_method 선택 기준

```
innodb_flush_method 옵션별 동작:

fsync (기본값):
  읽기: Page Cache → (없으면 디스크) → Buffer Pool
  쓰기: Buffer Pool → Page Cache → fsync() → 디스크
  특징: 이중 캐싱 발생, 안정적

O_DIRECT:
  읽기: 디스크 → Buffer Pool (Page Cache 우회)
  쓰기: Buffer Pool → O_DIRECT → 디스크 (Page Cache 우회)
  특징: 이중 캐싱 없음, Buffer Pool 크기 = 실제 캐시 크기

O_DSYNC:
  읽기: Page Cache 사용
  쓰기: Page Cache → 동기 디스크 쓰기 (데이터만, 메타데이터 제외)
  특징: 성능-내구성 중간 타협

선택 기준:
  물리 메모리 충분: O_DIRECT (Buffer Pool 제어 정확)
  메모리 제한 환경: fsync (Page Cache가 Buffer Pool 보조)
  SSD + 최대 성능: O_DIRECT + innodb_buffer_pool_size=70~80%
```

### Elasticsearch의 mmap 전략

```
Lucene 인덱스 파일 접근:
  기본: NIOFSDirectory (read() 시스템 콜)
  권장: MMapDirectory (mmap())

MMapDirectory 장점:
  → Lucene 세그먼트 파일을 가상 주소에 직접 매핑
  → Page Cache = Lucene 인덱스 캐시 (자동 관리)
  → GC 대상 아님 (JVM Heap 아닌 Direct Memory)
  → OS가 LRU로 자동 Eviction

Elasticsearch 권장 설정:
  JVM Heap: 물리 메모리의 50% 이하
  나머지 50%: OS Page Cache (Lucene 인덱스 캐시로 활용)

  예시: 64GB 서버
  -Xmx31g (31GB Heap)
  나머지 33GB: Page Cache (Lucene 인덱스 캐싱)
```

---

## 💻 실전 실험

### 실험 1: mmap vs read() 성능 비교

```c
/* mmap_vs_read.c */
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <time.h>
#include <string.h>

#define FILE_PATH "/tmp/bench_file"
#define FILE_SIZE (512 * 1024 * 1024)  // 512MB

long time_ms() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000 + ts.tv_nsec / 1000000;
}

int main() {
    // 테스트 파일 생성
    int fd = open(FILE_PATH, O_RDWR | O_CREAT | O_TRUNC, 0644);
    char *zeros = calloc(4096, 1);
    for (int i = 0; i < FILE_SIZE / 4096; i++) write(fd, zeros, 4096);
    close(fd);

    // Page Cache 비우기
    system("echo 3 > /proc/sys/vm/drop_caches 2>/dev/null");

    // read() 방식
    long start = time_ms();
    fd = open(FILE_PATH, O_RDONLY);
    char *buf = malloc(4096);
    long sum = 0;
    while (read(fd, buf, 4096) > 0) sum += buf[0];
    close(fd);
    printf("read():  %ld ms (sum=%ld)\n", time_ms() - start, sum);

    system("echo 3 > /proc/sys/vm/drop_caches 2>/dev/null");

    // mmap() 방식
    start = time_ms();
    fd = open(FILE_PATH, O_RDONLY);
    char *addr = mmap(NULL, FILE_SIZE, PROT_READ, MAP_PRIVATE, fd, 0);
    madvise(addr, FILE_SIZE, MADV_SEQUENTIAL);
    sum = 0;
    for (size_t i = 0; i < FILE_SIZE; i += 4096) sum += addr[i];
    munmap(addr, FILE_SIZE);
    close(fd);
    printf("mmap():  %ld ms (sum=%ld)\n", time_ms() - start, sum);

    free(buf); free(zeros);
    return 0;
}
```

```bash
$ gcc mmap_vs_read.c -o mmap_vs_read
$ sudo ./mmap_vs_read
# 예상 출력:
# read():  1200 ms (sum=0)   ← 시스템 콜 오버헤드
# mmap():   850 ms (sum=0)   ← Page Fault만, 복사 없음
# 두 번째 실행 (Page Cache 히트):
# read():   300 ms            ← 메모리 복사만
# mmap():   150 ms            ← 직접 메모리 접근
```

### 실험 2: O_DIRECT 정렬 요구사항

```c
/* direct_io_test.c */
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>

int main() {
    int fd = open("/tmp/direct_test", O_RDWR | O_CREAT | O_DIRECT, 0644);
    if (fd < 0) { perror("open"); return 1; }

    // 정렬된 버퍼 (posix_memalign 사용)
    void *aligned_buf;
    posix_memalign(&aligned_buf, 4096, 4096);  // 4096 정렬

    // 정렬된 쓰기 (성공)
    memset(aligned_buf, 'A', 4096);
    ssize_t n = write(fd, aligned_buf, 4096);
    printf("정렬된 쓰기: %zd bytes\n", n);

    // 정렬 안 된 버퍼 (실패)
    char *unaligned = malloc(5000) + 1;  // 1byte 오정렬
    lseek(fd, 0, SEEK_SET);
    n = write(fd, unaligned, 4096);
    if (n == -1) printf("오정렬 쓰기 실패: %s\n", strerror(errno));
    // 출력: 오정렬 쓰기 실패: Invalid argument

    free(aligned_buf);
    close(fd);
    return 0;
}
```

### 실험 3: fincore로 Page Cache 상태 확인

```bash
# fincore: 파일이 Page Cache에 얼마나 올라와 있는지 확인
$ apt-get install -y linux-tools-common || pip install fincore

# MySQL 데이터 파일의 Page Cache 상태
$ fincore /var/lib/mysql/ibdata1
filename        size        cached  cached%
ibdata1         1073741824  524288000  48.8%  ← 50% Page Cache에 있음

# O_DIRECT 설정 후 확인
$ fincore /var/lib/mysql/ibdata1
filename        size   cached  cached%
ibdata1         1073741824  0   0.0%  ← Page Cache 없음 (O_DIRECT)
```

---

## 📊 성능/비용 비교

| 방식 | 복사 횟수 | 시스템 콜 | Page Cache | 적합한 워크로드 |
|------|---------|---------|-----------|--------------|
| read() | 1회 | 매번 | 사용 | 소용량, 단순 |
| mmap() | 0회 | 최초 1회 | 사용 | 대용량, 랜덤 접근 |
| O_DIRECT | 1회 (DMA 직접) | 매번 | 미사용 | 자체 캐시 있는 DB |

**mmap vs read() 실제 차이가 나는 시나리오**:

```
mmap이 유리한 경우:
  - 파일 크기 >> 시스템 콜 오버헤드 (대용량 파일)
  - 동일 파일의 여러 프로세스 공유 (코드 공유 라이브러리)
  - 랜덤 접근 패턴 (파일의 임의 위치 접근)
  - Lucene처럼 인덱스 파일을 자주 참조하는 경우

read()가 유리한 경우:
  - 소용량 파일 (시스템 콜 오버헤드가 상대적으로 적음)
  - 순차 읽기 (read()의 Read-Ahead가 효과적)
  - 한 번만 읽는 경우
```

---

## ⚖️ 트레이드오프

**mmap의 함정: 메모리 오버커밋**

mmap()으로 파일을 매핑하면 가상 주소 공간을 많이 사용하게 된다. 32비트 시스템에서는 가상 주소 고갈 문제가 있었다. 64비트에서는 가상 주소 공간이 충분하지만, `vm.overcommit_memory` 설정에 따라 mmap()이 실패할 수 있다.

**O_DIRECT의 복잡성**

O_DIRECT는 강력하지만 구현 복잡도가 높다. 정렬 요구사항을 직접 관리해야 하고, Write Combining이 없어 작은 쓰기가 비효율적이다. MySQL이나 RocksDB 같은 성숙한 DB 엔진은 이 복잡성을 내부적으로 처리하지만, 직접 구현하기는 까다롭다.

---

## 📌 핵심 정리

```
mmap():
  파일을 가상 주소 공간에 직접 매핑
  복사 없음 (Page Cache 물리 페이지를 PTE에 직접 연결)
  첫 접근 시 Page Fault → 이후 시스템 콜 없음
  Elasticsearch(Lucene), 공유 라이브러리 로딩에 활용

O_DIRECT:
  Page Cache 완전 우회
  DMA → 애플리케이션 버퍼 직접
  정렬 요구: 버퍼/오프셋/크기 모두 섹터 크기 배수
  MySQL O_DIRECT, RocksDB Direct I/O에 활용

MySQL flush_method 선택:
  O_DIRECT: Buffer Pool = 유일한 캐시 (정확한 메모리 제어)
  fsync: Page Cache + Buffer Pool 이중 캐싱 (메모리 예측 어려움)

madvise() 힌트:
  MADV_SEQUENTIAL: Read-Ahead 요청 (순차 읽기)
  MADV_RANDOM:     Read-Ahead 끔 (랜덤 접근)
  MADV_WILLNEED:   미리 Page Cache에 로딩
  MADV_DONTNEED:   해당 영역 즉시 해제 허용
```

---

## 🤔 생각해볼 문제

**Q1.** Elasticsearch 클러스터에서 JVM Heap을 64GB 서버에서 60GB로 설정했다. 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

두 가지 문제가 발생한다. 첫째, Page Cache가 4GB 이하로 줄어 Lucene 인덱스 파일의 캐시 히트율이 급격히 낮아진다. `mmap()`으로 매핑된 인덱스 파일 접근마다 디스크 I/O가 발생해 검색 응답 시간이 수십~수백 배 증가한다. 둘째, 60GB Heap에서 G1GC가 Full GC를 실행하면 STW 시간이 수십 초에 달할 수 있다. Elasticsearch는 JVM Heap을 물리 메모리의 50%, 최대 31GB(G1GC 효율 한도)로 제한하도록 강력히 권장한다.

</details>

**Q2.** `mmap()`으로 매핑한 파일에 다른 프로세스가 write()로 내용을 변경했다. mmap()으로 매핑한 프로세스는 이 변경을 즉시 볼 수 있는가?

<details>
<summary>해설 보기</summary>

**`MAP_SHARED`로 매핑한 경우 즉시 볼 수 있다.** 두 접근 모두 같은 Page Cache 물리 페이지를 참조하기 때문이다. write()로 Page Cache가 수정되면 mmap() 주소 공간에도 즉시 반영된다. 반면 `MAP_PRIVATE`로 매핑한 경우 CoW 방식으로 독립된 복사본을 만들어, 다른 프로세스의 변경이 보이지 않는다. 공유 메모리 IPC 구현에 `mmap()` + `MAP_SHARED`를 활용하는 이유가 이것이다.

</details>

**Q3.** RocksDB에서 Block Cache를 1GB로 설정하고 O_DIRECT를 활성화했다. O_DIRECT를 비활성화하면 실제 캐시 용량이 왜 2배 이상이 되는가?

<details>
<summary>해설 보기</summary>

O_DIRECT 비활성화 시 RocksDB가 파일을 읽을 때 OS Page Cache에 자동으로 캐싱된다. 결과적으로 Block Cache(1GB) + OS Page Cache(OS가 할당하는 만큼, 수 GB)가 동시에 동일 데이터를 캐싱한다. Block Cache의 Eviction 정책(LRU 등)과 무관하게 Page Cache에 데이터가 남아있어 Block Cache가 Evict해도 실제로 메모리에서 제거되지 않는다. 이 상태에서 메모리 사용량은 예측이 어렵고, Block Cache 설정이 의미를 잃는다. O_DIRECT 활성화로 Block Cache가 유일한 캐시가 되어야 정확한 메모리 제어가 가능하다.

</details>

---

**[⬅️ 이전: Page Cache](./03-page-cache.md)** | **[홈으로 🏠](../README.md)** | **[다음: 메모리 할당기 ➡️](./05-memory-allocator.md)**
