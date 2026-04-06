# 01. VFS — 리눅스가 파일 시스템을 추상화하는 방식

## 🎯 핵심 질문

- VFS(Virtual File System)란 무엇이고, 왜 필요한가?
- `inode`, `dentry`, `file` 오브젝트는 각각 무엇을 담당하는가?
- `cat /proc/meminfo`와 `cat /var/log/app.log`가 같은 `read()` 시스템 콜로 동작하는 원리는?
- `/proc`와 `/sys`가 디스크가 없는데도 파일처럼 보이는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**`/proc/<pid>/status`를 읽으면 커널 내부 데이터를 파일처럼 접근한다.** 이것이 가능한 이유는 VFS가 커널 자료구조를 파일 인터페이스로 노출하기 때문이다. `strace`, `jstack`, `free -h` 같은 진단 도구들이 모두 `/proc`를 통해 동작한다. VFS 레이어를 이해하면 이 도구들이 왜 그렇게 설계됐는지 보인다.

**tmpfs를 활용해 Redis의 RDB 파일을 메모리에 저장하는 최적화가 있다.** `/dev/shm`은 tmpfs 기반 파일 시스템으로, 파일처럼 쓰고 읽지만 실제로는 RAM에만 존재한다. VFS가 파일 시스템 구현을 추상화하기 때문에 애플리케이션 코드 변경 없이 마운트 포인트만 바꿔 동작을 바꿀 수 있다.

**Docker 컨테이너가 호스트 파일 시스템을 격리하는 방식의 핵심이 VFS다.** Mount Namespace + OverlayFS 조합이 컨테이너에게 독립된 파일 시스템 뷰를 제공한다. VFS가 마운트 포인트별로 다른 파일 시스템을 연결하는 원리를 알면 이 격리가 어떻게 구현되는지 이해할 수 있다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: /proc 파일을 일반 파일처럼 취급
# "파일 크기가 0인데 왜 내용이 있지?"
$ ls -la /proc/meminfo
-r--r--r-- 1 root root 0 Jan  1 00:00 /proc/meminfo
#                      ↑ 크기 0? 하지만 내용은 수 KB!
# → 이것은 실제 파일이 아님
# → 읽기 시 커널이 그 순간의 메모리 상태를 동적으로 생성

# 실수 2: tmpfs를 "임시" 저장소로 오해
$ df -h /dev/shm
tmpfs            15Gi     0  15Gi   0% /dev/shm
# "쓰면 디스크에 저장되나?"
# → RAM에만 저장됨 (재부팅 시 사라짐)
# → 그러나 ext4처럼 동일한 파일 인터페이스 사용 가능

# 실수 3: 서로 다른 파일 시스템 존재를 모름
$ df -T
Filesystem     Type
/dev/sda1      ext4
tmpfs          tmpfs
proc           proc
sysfs          sysfs
# "이게 다 다른 건가?" → VFS가 통일된 인터페이스 제공
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# /proc 파일 시스템 활용: 실시간 커널 상태 조회
$ cat /proc/meminfo             # 메모리 현황 (동적 생성)
$ cat /proc/net/tcp             # TCP 연결 상태 (동적 생성)
$ cat /proc/$(pgrep redis)/maps # 프로세스 가상 주소 공간 (동적 생성)

# /sys 파일 시스템: 커널 파라미터 실시간 변경
$ echo never > /sys/kernel/mm/transparent_hugepage/enabled
$ cat /sys/block/sda/queue/scheduler  # I/O 스케줄러 확인/변경

# 마운트된 파일 시스템 전체 확인
$ findmnt --output TARGET,SOURCE,FSTYPE,OPTIONS
TARGET           SOURCE    FSTYPE  OPTIONS
/                /dev/sda1 ext4    rw,relatime
/proc            proc      proc    rw,nosuid
/sys             sysfs     sysfs   rw,nosuid
/dev/shm         tmpfs     tmpfs   rw,nosuid
/sys/fs/cgroup   cgroup2   cgroup2 rw

# tmpfs를 Redis RDB 저장소로 활용
$ mount -t tmpfs -o size=4g tmpfs /var/lib/redis-shm
# → RDB 저장/로드가 디스크 I/O 없이 메모리 속도로 처리
```

---

## 🔬 내부 동작 원리

### VFS의 역할과 위치

```
[애플리케이션]
  open("/var/log/app.log")
  read(fd, buf, 4096)
  open("/proc/meminfo")
  read(fd, buf, 4096)
         ↓ 동일한 시스템 콜!
[VFS (Virtual File System)]   ← 이 레이어가 통일된 인터페이스 제공
         ↓ 파일 시스템 유형에 따라 분기
  ┌──────┬──────┬──────┬──────┬──────┐
  │ ext4 │ XFS  │tmpfs │procfs│sysfs │
  └──────┴──────┴──────┴──────┴──────┘
         ↓              ↓
    [디스크 I/O]    [커널 메모리]
```

VFS는 파일 시스템 종류에 무관하게 동일한 `open()`, `read()`, `write()`, `close()` 인터페이스를 제공한다. 각 파일 시스템은 VFS가 정의한 오퍼레이션 구조체(`file_operations`, `inode_operations`)를 구현한다.

### VFS의 핵심 오브젝트 4가지

```
1. superblock — 마운트된 파일 시스템 전체 정보
   ┌─────────────────────────────────┐
   │ s_type: ext4                    │
   │ s_blocksize: 4096               │
   │ s_magic: 0xEF53 (ext4 식별자)     │
   │ s_op: →superblock_operations    │ ← fsync, statfs 등 구현 함수 포인터
   │ s_root: →dentry (루트 디렉토리)    │
   └─────────────────────────────────┘

2. inode — 파일 메타데이터 (이름 제외)
   ┌─────────────────────────────────┐
   │ i_ino: 12345 (inode 번호)        │
   │ i_mode: 0644 (권한)              │
   │ i_size: 4096 (파일 크기)          │
   │ i_uid, i_gid: 소유자              │
   │ i_atime, i_mtime, i_ctime       │
   │ i_op: →inode_operations         │ ← lookup, create, link 등
   │ i_fop: →file_operations         │ ← read, write, mmap 등
   │ i_mapping: →address_space       │ ← Page Cache 연결
   └─────────────────────────────────┘

3. dentry — 디렉토리 엔트리 (이름 ↔ inode 매핑)
   ┌─────────────────────────────────┐
   │ d_name: "app.log"               │
   │ d_inode: →inode                 │
   │ d_parent: →dentry (부모 디렉토리)  │
   │ d_subdirs: [자식 dentry 목록]     │
   └─────────────────────────────────┘
   → dentry 캐시(dcache)에 최근 경로 조회 결과 캐싱
   → /var/log/app.log 조회 = "var" + "log" + "app.log" 3번 dentry 탐색

4. file — 열린 파일 인스턴스 (fd당 1개)
   ┌─────────────────────────────────┐
   │ f_pos: 4096 (현재 읽기 오프셋)      │
   │ f_flags: O_RDONLY               │
   │ f_mode: FMODE_READ              │
   │ f_inode: →inode                 │
   │ f_op: →file_operations          │
   └─────────────────────────────────┘
   → 같은 파일을 여러 프로세스가 열면 file 오브젝트 여러 개
   → 각자 독립적인 f_pos (읽기 위치)를 가짐
```

### open() 시스템 콜의 VFS 처리 흐름

```
open("/var/log/app.log", O_RDONLY):

1. 경로 파싱: "/" → "var" → "log" → "app.log"
2. 각 단계에서 dentry 캐시(dcache) 조회
   히트: dentry에서 inode 즉시 획득
   미스: 디스크에서 디렉토리 엔트리 읽기 → dentry 캐시 추가
3. 최종 inode 획득 → 권한 확인 (i_mode vs 프로세스 uid/gid)
4. file 오브젝트 생성 (f_pos = 0)
5. 프로세스 파일 디스크립터 테이블에 file 등록
6. fd 번호 반환

procfs 파일 open("/proc/meminfo"):
1~3. 동일 (procfs의 dentry/inode 조회)
4. file 오브젝트 생성
   → f_op = &proc_meminfo_fops  ← procfs 전용 함수 포인터
5. fd 반환

read(proc_fd, buf, 4096):
   → file.f_op->read() 호출
   → proc_meminfo_read() 실행
   → 그 순간의 /proc/meminfo/meinfo 구조체를 텍스트로 직렬화
   → 버퍼에 복사 후 반환
   → 디스크 I/O 없음!
```

### /proc와 /sys의 특별한 구현

```
procfs (/proc):
  마운트: mount -t proc proc /proc
  디스크: 없음 (가상)
  inode: 읽기 시 동적으로 커널 자료구조를 파일 내용으로 변환
  쓰기 가능한 파일: 커널 파라미터 변경 (예: /proc/sys/vm/*)
  주요 파일:
    /proc/<pid>/status   → task_struct의 일부 필드
    /proc/<pid>/maps     → mm_struct의 VMA 목록
    /proc/<pid>/fd/      → 파일 디스크립터 테이블

sysfs (/sys):
  마운트: mount -t sysfs sysfs /sys
  디스크: 없음 (가상)
  목적: 커널 오브젝트(장치, 드라이버, cgroups)를 파일 계층으로 표현
  쓰기로 커널 상태 변경:
    /sys/block/sda/queue/scheduler    → I/O 스케줄러 변경
    /sys/fs/cgroup/memory/<id>/memory.limit_in_bytes → 메모리 한도
    /sys/kernel/mm/transparent_hugepage/enabled      → THP 설정

tmpfs (/dev/shm, /tmp):
  마운트: tmpfs
  디스크: 없음 (RAM + swap)
  ext4와 동일한 파일 인터페이스
  재부팅 시 사라짐
  Docker 컨테이너: /tmp, /run이 tmpfs
```

---

## 💻 실전 실험

### 실험 1: VFS 레이어 동작 확인 (strace)

```bash
# 서로 다른 파일 시스템에 대해 동일한 시스템 콜 확인
$ strace -e trace=openat,read,close cat /proc/meminfo 2>&1 | head -10
openat(AT_FDCWD, "/proc/meminfo", O_RDONLY) = 3
read(3, "MemTotal:       16384000 kB\nMemF"..., 65536) = 1234
close(3)                                = 0

$ strace -e trace=openat,read,close cat /var/log/syslog 2>&1 | head -10
openat(AT_FDCWD, "/var/log/syslog", O_RDONLY) = 3
read(3, "Jan  1 00:00:00 host kernel: ...", 65536) = 4096
close(3)                                = 0

# 동일한 openat + read + close 패턴!
# VFS가 파일 시스템 차이를 완전히 추상화
```

### 실험 2: dentry 캐시 효과 측정

```bash
# dentry 캐시 비우기
$ echo 2 > /proc/sys/vm/drop_caches  # inode + dentry 캐시 삭제

# 1차 경로 조회 (dentry 캐시 미스 - 디스크 탐색)
$ time find /usr/lib -name "*.so" > /dev/null
real 0m2.134s  ← 디스크 탐색 필요

# 2차 경로 조회 (dentry 캐시 히트)
$ time find /usr/lib -name "*.so" > /dev/null
real 0m0.087s  ← 24배 빠름! dentry 캐시 효과

# dentry 캐시 사용량 확인
$ cat /proc/meminfo | grep -i dentry
# (Slab 내에 포함됨)
$ slabtop -o | grep dentry
  12345  12000  97%  0.19K  dentry ← dentry 캐시 엔트리 수
```

### 실험 3: /proc 동적 생성 확인

```bash
# /proc 파일 내용이 동적으로 생성됨을 확인
$ PID=$(java -jar app.jar &; echo $!)

# 1초 간격으로 메모리 사용량 변화 관찰
$ while true; do
    date
    cat /proc/$PID/status | grep VmRSS
    sleep 1
  done
# → 매번 다른 값이 출력됨 (파일이 아닌 커널 상태 반영)

# /proc 파일의 크기가 0인 이유
$ ls -la /proc/$PID/status
-r--r--r-- 1 app app 0  ← 크기 0 (하지만 내용 있음)
# stat()이 파일 크기를 0으로 반환하지만
# read() 호출 시 커널이 내용을 동적 생성
```

---

## 📊 성능/비용 비교

| 파일 시스템 | 저장소 | 속도 | 영속성 | 주요 용도 |
|------------|-------|-----|-------|---------|
| ext4/XFS | SSD/HDD | 중간 | 영구 | 일반 데이터 |
| tmpfs | RAM(+swap) | 매우 빠름 | 비영구 | 임시 파일, 캐시 |
| procfs | 없음(동적) | 즉시 | N/A | 커널 상태 조회 |
| sysfs | 없음(동적) | 즉시 | N/A | 커널 파라미터 |
| OverlayFS | 기반 FS | 기반 FS | 기반 FS | Docker 레이어 |

**dentry 캐시 효과**:

```
경로 /var/lib/mysql/data/ibdata1 조회 시:
  캐시 미스: 각 디렉토리 단계별 디스크 읽기 (4~5번)
  캐시 히트: 메모리에서 즉시 inode 번호 획득
  차이: 수 ms vs 수 μs (1000배+)

MySQL 쿼리 처리 중 파일 열기:
  ibdata1, ib_logfile0 등 동일 파일을 반복 접근
  → dentry 캐시로 경로 탐색 비용 거의 없음
  → inode 캐시로 메타데이터도 메모리에서 즉시
```

---

## ⚖️ 트레이드오프

**VFS 추상화의 비용**

VFS는 편리한 통일 인터페이스를 제공하지만, 모든 파일 접근마다 VFS 레이어를 거친다. 간단한 read() 하나에도 dentry 조회, inode 권한 확인, file_operations 디스패치 등의 오버헤드가 있다. 고성능 스토리지(NVMe)에서는 이 소프트웨어 레이어 비용이 의미 있는 수준이 된다. `io_uring`은 이 오버헤드를 줄이기 위해 설계됐다.

**tmpfs: 속도 vs 내구성**

tmpfs는 메모리 속도의 파일 I/O를 제공하지만 서버 재시작 시 데이터가 사라진다. Redis의 `/dev/shm` 활용은 RDB 스냅샷을 빠르게 저장하는 데 유용하지만, 재시작 후에는 AOF나 원격 복제본에 의존해야 한다. 이 트레이드오프를 명확히 이해하고 사용해야 한다.

---

## 📌 핵심 정리

```
VFS:
  모든 파일 시스템(ext4, tmpfs, procfs, sysfs)을 동일한
  open()/read()/write() 인터페이스로 추상화

핵심 오브젝트:
  superblock → 마운트된 파일 시스템 전체 정보
  inode      → 파일 메타데이터 (이름 제외)
  dentry     → 파일명 ↔ inode 매핑, 경로 탐색 캐시
  file       → 열린 파일 인스턴스 (fd당 1개, 오프셋 포함)

/proc: 커널 자료구조를 파일로 노출 (디스크 없음, 동적 생성)
/sys:  커널 오브젝트를 파일 계층으로 표현 (파라미터 변경 가능)
tmpfs: RAM 기반 파일 시스템 (ext4 인터페이스, 비영구)

핵심 진단 명령어:
  findmnt                    → 마운트된 파일 시스템 트리
  df -T                      → 파일 시스템 종류 확인
  slabtop -o | grep dentry   → dentry 캐시 사용량
  echo 2 > /proc/sys/vm/drop_caches → inode+dentry 캐시 삭제 (테스트용)
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 파일을 두 프로세스가 동시에 열면 inode, dentry, file 오브젝트는 각각 몇 개 생성되는가?

<details>
<summary>해설 보기</summary>

inode: **1개** (파일 자체의 메타데이터, 공유), dentry: **1개** (경로 ↔ inode 매핑, 캐시 공유), file: **2개** (프로세스마다 독립적인 오프셋과 플래그 필요). 두 프로세스가 각자 독립적인 읽기 위치(f_pos)를 가지는 것이 이것 때문이다. 단, `dup()`이나 `fork()`로 같은 file 오브젝트를 공유하면 오프셋도 공유된다.

</details>

**Q2.** Docker 컨테이너에서 `cat /proc/self/status`를 실행하면 호스트의 정보가 보이는가, 컨테이너의 정보가 보이는가?

<details>
<summary>해설 보기</summary>

**컨테이너의 정보가 보인다.** Docker는 PID Namespace를 사용해 컨테이너 내부 프로세스에게 독립된 PID를 부여한다. `/proc`는 커널이 현재 네임스페이스 컨텍스트에 맞게 마운트되므로, 컨테이너 내에서 `/proc`를 보면 컨테이너 내부 프로세스들만 보인다. 단, `--pid=host` 옵션으로 컨테이너를 실행하면 호스트의 `/proc`에 접근할 수 있다.

</details>

**Q3.** `/proc/sys/vm/swappiness`에 값을 쓰면 어떤 일이 일어나는가? 이것이 일반 파일 쓰기와 어떻게 다른가?

<details>
<summary>해설 보기</summary>

VFS의 write() 시스템 콜이 procfs의 write 핸들러를 호출하고, 이 핸들러가 커널 변수 `vm_swappiness`의 값을 변경한다. 디스크에 아무것도 쓰지 않는다. 재부팅하면 원래 값으로 돌아간다. 일반 파일 쓰기는 Page Cache를 통해 디스크에 데이터를 기록하지만, procfs/sysfs 쓰기는 해당 파일에 연결된 커널 함수를 호출하는 트리거 역할을 한다. `/etc/sysctl.conf`로 영구 설정하면 부팅 시 이 값들을 다시 쓰는 것이다.

</details>

---

**[홈으로 🏠](../README.md)** | **[다음: 디스크 I/O 스택 ➡️](./02-disk-io-stack.md)**
