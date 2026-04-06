# 05. inode와 파일 시스템 구조 — 파일 삭제의 실체

## 🎯 핵심 질문

- inode는 무엇을 저장하고, 파일 이름은 왜 inode에 없는가?
- `rm`으로 파일을 삭제했는데 디스크 공간이 줄지 않는 이유는?
- `lsof +L1`이 "삭제됐지만 아직 열려있는 파일"을 어떻게 찾는가?
- inode 소진(`No space left on device`)이 발생하는 이유와 해결 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**로그 로테이션 후 디스크 공간이 줄지 않는다.** `logrotate`가 오래된 로그 파일을 삭제했는데 `df`에서 공간이 그대로다. Java 애플리케이션이 오래된 로그 파일 fd를 계속 열고 있기 때문이다. 링크 카운트가 0이 됐지만 fd가 살아있으면 커널이 공간을 해제하지 않는다.

**`df`는 공간이 있는데 파일 생성이 안 된다.** inode 소진이다. ext4 파일 시스템은 포맷 시 inode 수를 미리 계산한다(기본: 바이트당 1개). 수백만 개의 작은 파일이 생성되면 inode가 먼저 고갈된다. Elasticsearch 인덱스가 많거나, 세션 파일, 임시 파일이 쌓이면 이 문제가 발생한다.

**하드링크가 백업 솔루션에서 어떻게 활용되는가?** 동일한 inode를 가리키는 여러 디렉토리 엔트리(하드링크)를 이용하면, 파일 내용을 복사 없이 "어제 백업"과 "오늘 백업"이 변경되지 않은 파일을 공유할 수 있다. rsync의 `--link-dest` 옵션이 이 원리를 사용한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: rm 후 디스크 공간이 해제됐다고 가정
$ rm /var/log/app/access.log.1
$ df -h /var/log
Filesystem  Size  Used Avail Use%
/dev/sda1   100G   80G   20G  80%  ← 변화 없음!

# "왜 안 줄지?" → Java 프로세스가 파일 핸들 유지 중
$ lsof +L1 | grep "access.log"
java  1234  1234  18u  REG  8,1  0  12345 /var/log/app/access.log.1 (deleted)
#                                                                      ↑ 삭제됐지만 열려있음

# 실수 2: inode 소진을 디스크 부족으로 오해
$ df -h
Filesystem  Size  Used Avail Use% Mountpoint
/dev/sda1   100G   50G   50G  50% /          ← 공간 있음
$ touch newfile
touch: cannot touch 'newfile': No space left on device  ← 에러!

# df -i로 확인
$ df -i
Filesystem    Inodes  IUsed   IFree IUse% Mountpoint
/dev/sda1   6553600 6553600       0  100% /  ← inode 소진!
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 삭제됐지만 열려있는 파일 찾기 (공간 점유 중)
$ lsof +L1
# +L1: 링크 카운트 < 1인 파일 (삭제됐지만 fd 존재)
COMMAND  PID  USER   FD  TYPE DEVICE SIZE/OFF  NLINK NODE NAME
java    1234  app    18u  REG   8,1  524288000     0  12345 /var/log/app.log (deleted)
# → 500MB를 점유한 삭제된 로그 파일!

# 해결 방법 1: 해당 프로세스 재시작 (fd 해제)
$ kill -USR1 $PID  # graceful reload (애플리케이션 설정에 따라)

# 해결 방법 2: /proc로 fd 직접 비우기 (재시작 없이)
$ cat /dev/null > /proc/1234/fd/18  # fd 내용을 비워서 공간 해제
# 주의: 애플리케이션이 계속 이 fd에 쓰므로 빠르게 다시 찰 수 있음

# inode 소진 진단 및 예방
$ df -i  # inode 사용량 확인
$ find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -20
# 어느 디렉토리에 파일이 많은지 확인

# 작은 파일 많은 경우 ext4 재포맷 (inode 수 늘리기)
$ mkfs.ext4 -i 4096 /dev/sdb  # 4KB당 inode 1개 (기본 16KB당 1개)
```

---

## 🔬 내부 동작 원리

### inode 구조

```
inode = 파일의 메타데이터 저장소 (이름 제외)

struct ext4_inode {
    __le16 i_mode;          /* 파일 유형 + 권한 (0644, 0755 등) */
    __le16 i_uid;           /* 소유자 UID */
    __le32 i_size_lo;       /* 파일 크기 (하위 32비트) */
    __le32 i_atime;         /* 마지막 접근 시간 */
    __le32 i_ctime;         /* 마지막 상태 변경 시간 */
    __le32 i_mtime;         /* 마지막 수정 시간 */
    __le16 i_links_count;   /* ★ 링크 카운트 (하드링크 수) */
    __le32 i_blocks_lo;     /* 할당된 블록 수 */
    __le32 i_block[15];     /* 데이터 블록 포인터 */
    /* ext4 extent tree 추가 필드 ... */
};

inode에 없는 것:
  → 파일 이름 (dentry에 있음)
  → 파일 경로 (dentry 트리에 있음)
  → 부모 디렉토리 정보

이유: 하나의 inode가 여러 이름(하드링크)을 가질 수 있기 때문
```

### 디렉토리 구조와 링크 카운트

```
디렉토리 = 이름 → inode 번호 매핑 테이블

/var/log/
  ├── app.log        → inode 12345
  ├── error.log      → inode 12346
  └── app.log.1      → inode 12345  (하드링크! 같은 inode)

inode 12345:
  i_links_count = 2  (app.log + app.log.1 두 곳에서 참조)
  size: 100MB
  data blocks: [블록 1, 블록 2, ...]

rm /var/log/app.log 실행:
  1. /var/log 디렉토리에서 "app.log" 엔트리 제거
  2. inode 12345의 i_links_count: 2 → 1
  3. 링크 카운트 > 0이므로 inode와 데이터 블록 유지
  4. /var/log/app.log.1 여전히 inode 12345 참조

rm /var/log/app.log.1 실행:
  1. /var/log 디렉토리에서 "app.log.1" 엔트리 제거
  2. inode 12345의 i_links_count: 1 → 0
  3. 링크 카운트 = 0이지만 열린 fd 있으면 아직 유지!
  4. 마지막 fd도 close() → 이때 실제로 블록 해제
```

### "삭제됐지만 공간 해제 안 됨" 메커니즘

```
파일 삭제 후 공간이 해제되는 조건:
  i_links_count == 0  (모든 하드링크 삭제)
  AND
  open_count == 0     (모든 프로세스가 fd 닫음)

[타임라인]
  t=0: java가 app.log(inode 12345, links=1, fd=18) 열어서 쓰는 중
  t=1: logrotate가 app.log를 rename → app.log.1
       → links 카운트 변화 없음 (같은 inode를 다른 이름으로 참조)
       → java의 fd=18은 여전히 inode 12345 가리킴
  t=2: logrotate가 app.log.1 삭제 (rm)
       → links: 1 → 0
       → 하지만 java의 fd=18이 살아있음 (open_count=1)
       → 커널이 inode와 데이터 블록 유지!
  t=3: java 재시작 → fd=18 close → open_count: 1 → 0
       → 이제 링크 0 + 오픈 0 → 블록 해제!
       → df에서 공간 감소 확인

/proc에서 확인:
  $ cat /proc/1234/fd/18 → (deleted) 파일 링크
  $ ls -la /proc/1234/fd/18
  lrwxrwxrwx fd/18 -> /var/log/app.log.1 (deleted)
```

### inode 소진과 파일 시스템 계획

```
ext4 inode 할당:
  포맷 시 결정: mkfs.ext4의 bytes-per-inode 옵션
  기본값: 16384 바이트당 inode 1개
  100GB 파티션: 100 * 1024 * 1024 * 1024 / 16384 = 6,553,600개

inode 소진 케이스:
  Case 1: 수백만 개의 작은 파일
    → Node.js npm 패키지 (node_modules/)
    → Elasticsearch 세그먼트 파일
    → 세션 파일 서버 (PHP session.save_path)

  Case 2: 이메일 서버 maildir 형식
    → 사용자마다 수천 개의 이메일 파일

  Case 3: 빌드 아티팩트
    → CI/CD 파이프라인의 임시 파일 누적

진단 명령어:
  $ df -i          → inode 사용량 퍼센트
  $ stat -f /      → 파일 시스템 통계 (inode 전체/사용/여유)
  $ find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn
    → 어느 디렉토리에 파일이 몰려있는지

해결 방법:
  단기: 불필요한 파일 삭제 (임시파일, 캐시, 로그)
  장기: 재포맷 시 bytes-per-inode 줄이기
    mkfs.ext4 -i 4096 /dev/sdb  (4KB당 1개 → 4배 많은 inode)
  또는: XFS 사용 (inode 동적 할당, 소진 없음)
```

### 하드링크 vs 심볼릭링크

```
하드링크:
  ln /data/original.txt /backup/copy.txt
  → 같은 inode를 두 개의 이름으로 참조
  → 어느 쪽 삭제해도 다른 쪽 유지
  → 파일 시스템 경계 불가 (같은 inode 테이블)
  → 디렉토리에 하드링크 불가 (순환 참조 위험)

심볼릭링크(symlink):
  ln -s /data/original.txt /backup/link.txt
  → 별도 inode 생성 (링크 자체가 파일)
  → 원본 삭제 시 "dangling symlink" (깨진 링크)
  → 파일 시스템 경계 가능 (다른 파티션 가리키기)
  → 디렉토리에도 가능

rsync --link-dest (증분 백업):
  오늘 백업: /backup/2024-01-02/file.txt → inode 99999 (새 파일)
  변경 없는 파일: /backup/2024-01-02/unchanged.txt → inode 11111
    → /backup/2024-01-01/unchanged.txt와 같은 inode!
    → 실제 복사 없음, 하드링크만 생성
    → 디스크 공간 절약
```

---

## 💻 실전 실험

### 실험 1: 링크 카운트와 삭제 동작 확인

```bash
# 파일 생성 후 inode 정보 확인
$ echo "test data" > /tmp/test_file.txt
$ stat /tmp/test_file.txt
  File: /tmp/test_file.txt
  Size: 10         Blocks: 8     IO Block: 4096  regular file
Device: 803h/2051d  Inode: 12345  Links: 1   ← 링크 카운트 1
Access: (0644/-rw-r--r--)  Uid: 1000  Gid: 1000

# 하드링크 추가
$ ln /tmp/test_file.txt /tmp/hardlink.txt
$ stat /tmp/test_file.txt | grep Links
Links: 2   ← 링크 카운트 2 (두 이름이 같은 inode 참조)

# inode 번호 동일 확인
$ ls -i /tmp/test_file.txt /tmp/hardlink.txt
12345 /tmp/test_file.txt
12345 /tmp/hardlink.txt   ← 동일 inode!

# 하나 삭제 → 링크 카운트만 감소
$ rm /tmp/test_file.txt
$ stat /tmp/hardlink.txt | grep Links
Links: 1   ← 여전히 1 (데이터 유지)
$ cat /tmp/hardlink.txt
test data  ← 내용 접근 가능
```

### 실험 2: lsof +L1로 삭제된 파일 fd 찾기

```bash
# 삭제됐지만 열려있는 파일 시뮬레이션
$ cat > /tmp/del_test.sh << 'EOF'
#!/bin/bash
exec 5>/tmp/held_file.txt  # fd 5로 파일 열기
echo "Process holding file: PID=$$"
rm /tmp/held_file.txt  # 삭제! 하지만 fd 5는 살아있음
echo "File deleted but fd 5 still open"
sleep 60  # fd 유지
EOF
$ bash /tmp/del_test.sh &

# 삭제됐지만 fd가 살아있는 파일 찾기
$ lsof +L1
COMMAND  PID  USER   FD  TYPE DEVICE  SIZE  NLINK  NODE NAME
bash    9999  user    5w   REG   8,1     0      0  54321 /tmp/held_file.txt (deleted)
#                                              ↑ 링크 카운트 0 = 삭제됨

# 해당 파일이 점유하는 디스크 공간 (SIZE 컬럼)
$ lsof +L1 | awk 'NR>1 {sum+=$7} END {print "총 점유:", sum/1024/1024, "MB"}'
```

### 실험 3: inode 소진 재현

```bash
# 작은 임시 파티션에서 inode 소진 재현
$ dd if=/dev/zero of=/tmp/small_fs.img bs=1M count=100
$ mkfs.ext4 /tmp/small_fs.img
# 기본 inode 수 확인
$ tune2fs -l /tmp/small_fs.img | grep "Inode count"
Inode count: 25600   ← 100MB / 4096 = 25,600개

$ mount -o loop /tmp/small_fs.img /mnt/test

# 수만 개의 빈 파일 생성
$ for i in $(seq 1 25601); do touch /mnt/test/file_$i; done
touch: cannot touch '/mnt/test/file_25601': No space left on device

# 공간은 남아있지만 inode 소진
$ df -h /mnt/test
Filesystem  Size  Used Avail Use%
/dev/loop0   93M  21M   65M  25%  ← 공간 75% 남음

$ df -i /mnt/test
Filesystem    Inodes  IUsed  IFree IUse%
/dev/loop0     25600  25600      0  100%  ← inode 소진!

$ umount /mnt/test
```

---

## 📊 성능/비용 비교

**파일 시스템별 inode 처리 방식**:

```
ext4:
  inode 포맷 시 고정 할당 (기본: 16KB당 1개)
  장점: 빠른 inode 조회
  단점: 소진 시 파일 시스템 재포맷 필요

XFS:
  inode 동적 할당 (파티션 공간이 있는 한 계속 생성)
  장점: inode 소진 없음
  단점: inode 테이블 관리 오버헤드

btrfs:
  동적 할당, Copy-On-Write 지원
  장점: 스냅샷, 중복 제거 지원
  단점: 쓰기 증폭, 복잡한 내부 구조

권장:
  소규모 파일 수백만 개: XFS (inode 소진 없음)
  일반 서버 워크로드: ext4 (안정성, 성숙도)
  스냅샷/백업 서버: btrfs
```

**하드링크 vs 복사 공간 비교**:

```
rsync 증분 백업 예시 (1TB 데이터, 일별 백업):
  단순 복사: 1TB * 30일 = 30TB
  --link-dest 하드링크:
    매일 변경 비율 1%: 0.01TB * 30일 = 0.3TB + 기본 1TB = 1.3TB
    공간 절약: 30TB → 1.3TB (95% 절약!)
```

---

## ⚖️ 트레이드오프

**inode 크기와 포맷 시 결정의 영향**

inode를 많이 만들면 inode 테이블 자체가 공간을 차지한다. 100GB 파티션에서 4KB당 inode 1개로 포맷하면 inode가 기본보다 4배 많지만, inode 테이블이 약 2~3GB를 차지한다. 반대로 inode가 너무 적으면 소진 문제가 발생한다. 서비스의 파일 패턴을 미리 파악해 적정 비율로 포맷하거나, 동적 할당을 지원하는 XFS를 선택하는 것이 낫다.

**열린 fd와 삭제된 파일의 수명 연장**

프로그램이 파일 fd를 계속 열어두면 삭제된 파일이 공간을 계속 점유한다. 이것이 버그인 경우(fd 누수)도 있지만, 의도적인 경우도 있다. 예를 들어 `mkstemp()`로 임시 파일을 만들고 즉시 `unlink()`해서 "이름 없는 임시 파일"로 사용하는 패턴이다. 프로세스 종료 시 fd가 자동으로 닫히면서 공간이 해제된다. 이 패턴은 안전하지만, 의도하지 않은 fd 누수와 구별해야 한다.

---

## 📌 핵심 정리

```
inode:
  파일 메타데이터 저장 (이름 제외)
  i_links_count: 하드링크 수
  링크 카운트 = 0 AND 열린 fd = 0 → 블록 해제

파일 삭제(unlink):
  디렉토리 엔트리 제거 + 링크 카운트 감소
  링크 카운트가 0이 돼도 열린 fd가 있으면 데이터 유지

로그 로테이션 후 공간 미해제 원인:
  프로세스가 이전 로그 파일 fd 보유 중
  → lsof +L1로 확인 → 재시작 또는 fd 비우기

inode 소진:
  df 공간 있지만 df -i 100% → 파일 생성 불가
  해결: 불필요 파일 삭제, 재포맷 (inode 수 조정), XFS 전환

핵심 진단 명령어:
  stat <file>        → inode 정보 (링크 카운트, 블록)
  ls -i              → inode 번호 표시
  lsof +L1           → 삭제됐지만 열린 파일
  df -i              → inode 사용량
  tune2fs -l <dev>   → ext4 파일 시스템 inode 설정
  find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn
                     → 파일 수 많은 디렉토리 찾기
```

---

## 🤔 생각해볼 문제

**Q1.** `rsync --link-dest`로 증분 백업을 구현했다. 원본 파일이 수정됐는데 어제 백업 파일도 같이 수정되는가?

<details>
<summary>해설 보기</summary>

**아니다.** 하드링크는 같은 inode를 가리키지만, 파일이 수정될 때 동작이 다르다. `rsync`는 파일 내용이 변경됐다면 새로운 inode를 생성해 오늘 백업에는 새 파일이 들어가고, 어제 백업의 하드링크는 이전 inode를 계속 가리킨다. 이것이 증분 백업이 효율적인 이유다. 단, 파일을 직접 `write()`로 수정하면(데이터 블록이 수정됨) 같은 inode를 가리키는 모든 하드링크에서 변경이 반영된다. `rsync`는 수정 시 새 파일을 만들어 rename하는 방식을 사용해 이 문제를 피한다.

</details>

**Q2.** `/proc/<pid>/fd/<n>`이 삭제된 파일을 가리키고 있다. 이 fd로 `read()`를 호출하면 여전히 데이터를 읽을 수 있는가?

<details>
<summary>해설 보기</summary>

**그렇다.** fd는 inode를 통해 데이터 블록을 참조한다. 파일이 디렉토리에서 삭제됐어도 inode와 데이터 블록이 메모리(Page Cache)와 디스크에 유지되는 한 `read()`로 데이터를 읽을 수 있다. 실제로 `lsof +L1`에서 발견된 파일의 내용을 읽으려면 `cat /proc/<pid>/fd/<n>`으로 접근할 수 있다. 이것이 컨테이너 포렌식에서 활용되는 기법이기도 하다.

</details>

**Q3.** MySQL의 InnoDB는 `ibdata1` 파일을 삭제할 수 없다. 만약 실수로 삭제했다면 MySQL이 재시작될 때 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

MySQL이 실행 중이라면 `ibdata1` inode와 데이터 블록이 MySQL 프로세스의 fd로 유지된다. MySQL을 재시작하지 않으면 당장은 영향이 없다. 하지만 MySQL을 재시작하면 fd가 닫히고 이때 inode 링크 카운트 0 + 열린 fd 0이 되어 디스크 블록이 실제로 해제된다. 재시작 시 `ibdata1`이 없으면 MySQL이 시작을 거부하거나, `innodb_force_recovery` 모드로 복구를 시도해야 한다. 중요 데이터가 유실될 수 있다. 실행 중에 `lsof | grep ibdata1`으로 파일이 여전히 열려있는지 확인하고, 즉시 백업 복사본으로 복구해야 한다.

</details>

---

**[⬅️ 이전: fsync와 내구성](./04-fsync-durability.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — 소켓과 커널 버퍼 ➡️](../network-stack/01-socket-kernel-buffer.md)**
