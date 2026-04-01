<div align="center">

# 🐧 Linux for Backend Deep Dive

**"리눅스에 배포하는 것과, 커널이 I/O와 메모리를 어떻게 관리해 내 애플리케이션 성능을 결정하는지 아는 것은 다르다"**

<br/>

> *"`top`으로 CPU를 보는 것 — 과 — `wa`(iowait)가 높을 때 Page Cache 히트율을 먼저 확인해야 한다는 것을 아는 것의 차이를 만드는 레포"*

MySQL이 Sequential Read에 최적화된 이유, Redis가 `fork()`로 스냅샷을 찍으면서도 쓰기를 받는 원리, Docker가 namespace와 cgroups 위에서 동작하는 방식, Spring WebFlux가 적은 스레드로 더 많은 요청을 처리하는 구조 —  
**이 모든 것의 기저에 있는 커널 동작 원리**를 `/proc` 파일 시스템과 `strace`로 직접 확인하며 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Linux](https://img.shields.io/badge/Linux-Kernel_5.x+-FCC624?style=flat-square&logo=linux&logoColor=black)](https://kernel.org/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Docs](https://img.shields.io/badge/Docs-38개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

백엔드 개발자에게 리눅스 관련 자료는 넘칩니다. 하지만 대부분은 **"명령어를 어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`innodb_buffer_pool_size`를 물리 메모리의 70~80%로 설정하세요" | InnoDB Buffer Pool과 OS Page Cache의 관계, `innodb_flush_method=O_DIRECT`가 이중 캐싱을 막는 원리, `free -h`에서 `buff/cache`가 실제로 무엇인지 |
| "Redis `BGSAVE`로 백업하세요" | `fork()` 직후 자식 프로세스가 부모의 가상 주소 공간을 공유하는 방식, Copy-On-Write로 쓰기 발생 시에만 페이지가 복사되는 원리, `/proc/<pid>/smaps`로 실제 메모리 증가를 측정 |
| "Spring WebFlux는 더 적은 스레드를 씁니다" | Tomcat의 스레드 1개 = 연결 1개 모델의 한계, Netty가 `epoll`로 이벤트를 감시하는 구조, `epoll_wait()` O(1) 이벤트 처리가 Blocking I/O의 O(N) 스캔과 다른 이유 |
| "Docker `--memory=512m`으로 메모리를 제한하세요" | `/sys/fs/cgroup/memory/docker/<id>/memory.limit_in_bytes`가 실제로 어디에 기록되는지, cgroups 한도 초과 시 커널 OOM Killer가 어떤 프로세스를 먼저 죽이는지 |
| "`iostat`으로 I/O를 확인하세요" | `await`(평균 I/O 대기 시간)과 `%util`(디스크 포화도)이 의미하는 것, Sequential vs Random I/O가 HDD/SSD에서 각각 다른 성능을 내는 커널 레벨 이유 |
| "Kafka는 Zero-copy로 빠릅니다" | `sendfile()` 시스템 콜이 커널 버퍼→유저 버퍼 복사 없이 데이터를 전송하는 방식, `strace`로 Kafka의 실제 시스템 콜 패턴을 확인 |
| 이론 나열 | 재현 가능한 실험 + `/proc` 파일 시스템 직접 확인 + `strace`로 시스템 콜 추적 + Docker Compose 기반 실험 환경 |

---

## 🔑 이 레포가 모든 백엔드 레포의 기저인 이유

```
redis-deep-dive의 BGSAVE 원리
  └→ 이 레포 Ch1 (fork + Copy-On-Write)

mysql-deep-dive의 innodb_buffer_pool_size 설정
  └→ 이 레포 Ch2 (Page Cache와 이중 캐싱)

kafka-deep-dive의 Zero-copy 성능
  └→ 이 레포 Ch4 (sendfile + Sequential I/O)

docker-deep-dive의 컨테이너 격리 원리
  └→ 이 레포 Ch6 (namespace + cgroups)

network-deep-dive의 고성능 서버 모델
  └→ 이 레포 Ch3 (epoll과 I/O 모델 진화)
```

> 💡 **학습 순서 권장**: 이 레포를 먼저 읽고, 이후 `docker-deep-dive` → `network-deep-dive` → `redis-deep-dive` → `kafka-deep-dive` 순으로 이어가면 각 레포의 "왜"가 자연스럽게 연결됩니다.

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Process](https://img.shields.io/badge/🔹_Ch1-프로세스와_스레드-FCC624?style=for-the-badge&logo=linux&logoColor=black)](./process-thread/01-process-address-space.md)
[![Memory](https://img.shields.io/badge/🔹_Ch2-메모리_관리-FCC624?style=for-the-badge&logo=linux&logoColor=black)](./memory-management/01-virtual-memory.md)
[![IO](https://img.shields.io/badge/🔹_Ch3-I/O_모델의_진화-FCC624?style=for-the-badge&logo=linux&logoColor=black)](./io-models/01-file-descriptor-syscall.md)
[![FS](https://img.shields.io/badge/🔹_Ch4-파일시스템과_디스크_I/O-FCC624?style=for-the-badge&logo=linux&logoColor=black)](./filesystem-disk-io/01-vfs.md)
[![Net](https://img.shields.io/badge/🔹_Ch5-네트워크_스택-FCC624?style=for-the-badge&logo=linux&logoColor=black)](./network-stack/01-socket-kernel-buffer.md)
[![Cgroups](https://img.shields.io/badge/🔹_Ch6-cgroups와_namespace-FCC624?style=for-the-badge&logo=linux&logoColor=black)](./cgroups-namespace/01-linux-namespace.md)
[![Perf](https://img.shields.io/badge/🔹_Ch7-성능_진단-FCC624?style=for-the-badge&logo=linux&logoColor=black)](./performance-diagnosis/01-cpu-analysis.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: 프로세스와 스레드 — 실행의 기본 단위

> **핵심 질문:** 프로세스와 스레드는 커널 레벨에서 어떻게 다른가? `fork()`의 Copy-On-Write가 Redis BGSAVE를 가능하게 하는 원리는? 컨텍스트 스위칭 비용이 실제 성능에 어떻게 영향을 주는가?

<details>
<summary><b>프로세스 주소 공간부터 CFS 스케줄러까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 프로세스란 무엇인가 — 주소 공간과 PCB](./process-thread/01-process-address-space.md) | 프로세스의 코드/데이터/힙/스택 세그먼트가 가상 주소 공간에 배치되는 방식, PCB(Process Control Block)가 저장하는 상태 정보, `/proc/<pid>/maps`로 실제 주소 공간 직접 확인 |
| [02. fork()와 exec() — Copy-On-Write의 실체](./process-thread/02-fork-exec-cow.md) | `fork()` 직후 자식이 부모의 페이지 테이블을 공유하는 방식, 쓰기 발생 시에만 페이지가 복사되는 Copy-On-Write 메커니즘, **Redis `BGSAVE`가 이 원리로 쓰기를 받으면서 스냅샷을 찍는 방식**, `exec()`으로 완전히 다른 프로그램을 실행하는 과정 |
| [03. 스레드 모델 — OS 스레드와 1:1 매핑](./process-thread/03-thread-model.md) | 스레드가 같은 주소 공간을 공유하면서 스택만 독립적으로 가지는 구조, 프로세스보다 생성 비용이 작은 이유(TLB 플러시 없음), `/proc/<pid>/task/`로 스레드 목록 확인, Java Virtual Thread가 OS 스레드 모델의 어떤 한계를 해결하는가 |
| [04. 컨텍스트 스위칭 비용 — CPU 캐시와 TLB 플러시](./process-thread/04-context-switching.md) | 컨텍스트 스위칭 시 CPU 레지스터 저장/복원, TLB(Translation Lookaside Buffer) 플러시, L1/L2 캐시 무효화가 발생하는 과정, Tomcat 스레드 풀 vs Netty 이벤트 루프의 컨텍스트 스위칭 비용 비교 |
| [05. 시그널과 IPC — SIGKILL vs SIGTERM](./process-thread/05-signal-ipc.md) | `SIGKILL`(즉시 종료, 핸들러 불가)과 `SIGTERM`(정상 종료, 핸들러 가능)의 커널 처리 차이, Graceful Shutdown이 `SIGTERM`에 의존하는 이유, Pipe/FIFO/Message Queue/Shared Memory 비교와 각각의 커널 구현 |
| [06. CFS 스케줄러 — 공정한 CPU 할당의 원리](./process-thread/06-cfs-scheduler.md) | CFS(Completely Fair Scheduler)가 `vruntime`(가상 실행 시간)으로 실행 순서를 결정하는 방식, `nice` 값이 CPU 가중치에 영향을 주는 계산식, CPU 어피니티로 특정 코어에 프로세스를 고정하는 방법, **cgroups의 CPU 쿼터 제한이 CFS 위에서 동작하는 방식** |

</details>

<br/>

### 🔹 Chapter 2: 메모리 관리 — 가상 주소 공간의 실체

> **핵심 질문:** 가상 메모리는 왜 존재하는가? Page Cache가 MySQL과 Kafka 성능의 열쇠인 이유는? OOM Killer는 어떤 기준으로 프로세스를 종료하는가?

<details>
<summary><b>가상 메모리부터 OOM Killer까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 가상 메모리 — Page Table과 TLB](./memory-management/01-virtual-memory.md) | 모든 프로세스가 독립된 가상 주소 공간을 가지는 이유, Page Table이 가상 주소→물리 주소를 변환하는 방식, TLB가 이 변환을 캐싱해 성능을 높이는 원리, `/proc/<pid>/maps`와 `/proc/<pid>/smaps`로 매핑 상태 확인 |
| [02. Page Fault — 물리 메모리가 할당되는 시점](./memory-management/02-page-fault.md) | Minor Page Fault(물리 페이지 매핑만 필요)와 Major Page Fault(디스크에서 읽기 필요)의 차이, `malloc()` 호출 시 바로 물리 메모리가 할당되지 않는 이유(Demand Paging), `/usr/bin/time -v`로 프로세스의 Page Fault 횟수 측정 |
| [03. Page Cache — 디스크 I/O를 메모리에서 처리하는 원리](./memory-management/03-page-cache.md) | 파일 읽기가 두 번째부터 빠른 이유(Page Cache 히트), `free -h`의 `buff/cache`가 실제로 무엇인지, **MySQL `innodb_flush_method=O_DIRECT`가 Page Cache를 우회해 이중 캐싱을 방지하는 방식**, **Kafka가 Page Cache를 적극 활용해 디스크 I/O 없이 Consumer에게 전달하는 구조** |
| [04. mmap과 Direct I/O — Page Cache 활용과 우회](./memory-management/04-mmap-direct-io.md) | `mmap()`이 파일을 가상 주소 공간에 매핑해 Page Cache와 연결하는 방식, `O_DIRECT`로 Page Cache를 완전히 우회해 애플리케이션이 직접 디스크를 관리하는 이유, MySQL이 상황에 따라 `fsync` vs `O_DIRECT`를 선택하는 기준 |
| [05. 메모리 할당기 — ptmalloc vs jemalloc vs tcmalloc](./memory-management/05-memory-allocator.md) | glibc 기본 할당기 `ptmalloc`의 arena 구조와 단편화 문제, **Redis가 `jemalloc`을 쓰는 이유**(단편화 감소, `MEMORY DOCTOR`로 확인), **JVM이 자체 Heap 관리를 하는 이유**, `tcmalloc`의 스레드 로컬 캐시 구조 |
| [06. OOM Killer — 프로세스 종료 기준과 예방](./memory-management/06-oom-killer.md) | 물리 메모리 고갈 시 커널이 `oom_score`를 계산해 종료 대상을 선택하는 방식, `oom_score_adj`로 보호 우선순위를 조정하는 방법, **컨테이너 cgroups 메모리 한도 초과 시 OOM이 발생하는 경위**, **JVM `-XX:MaxRAMPercentage`로 컨테이너 OOM을 예방하는 방법** |

</details>

<br/>

### 🔹 Chapter 3: I/O 모델의 진화 — Blocking에서 epoll까지

> **핵심 질문:** `read()` 한 번 호출 시 커널에서 무슨 일이 일어나는가? epoll은 select와 어떻게 다른가? Redis가 단일 스레드임에도 빠른 진짜 이유는?

<details>
<summary><b>파일 디스크립터부터 WebFlux vs Tomcat 비교까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 파일 디스크립터와 시스템 콜 — 커널의 관문](./io-models/01-file-descriptor-syscall.md) | fd(파일 디스크립터)가 커널의 파일 테이블 엔트리를 가리키는 방식, `read()`/`write()` 시스템 콜이 유저→커널 전환 후 하는 일, `strace`로 실제 시스템 콜을 추적해 애플리케이션의 I/O 패턴 분석, `/proc/<pid>/fd/`로 열린 fd 목록 확인 |
| [02. Blocking I/O — 프로세스가 잠드는 원리](./io-models/02-blocking-io.md) | `read()` 호출 시 데이터가 없으면 프로세스가 `TASK_INTERRUPTIBLE` 상태로 대기 큐에 들어가는 과정, 데이터 도착 시 인터럽트→프로세스 Wake-up→유저 공간 복사 전 과정, 스레드 1개 = 연결 1개 모델에서 10,000 동시 연결이 왜 문제인가 |
| [03. Non-Blocking I/O와 Polling — busy-wait의 함정](./io-models/03-nonblocking-polling.md) | `O_NONBLOCK`으로 `read()`가 데이터 없을 때 `EAGAIN`을 반환하는 방식, 애플리케이션이 루프로 반복 확인(busy-wait)할 때 CPU가 낭비되는 이유, `strace`로 `EAGAIN` 반환 횟수를 측정하는 실험 |
| [04. I/O Multiplexing — select/poll의 한계](./io-models/04-io-multiplexing-select-poll.md) | `select()`/`poll()`이 하나의 스레드로 여러 fd를 감시하는 방식, 매번 전체 fd 목록을 커널에 전달하는 `O(N)` 스캔의 한계, fd 최대 개수 제한(`FD_SETSIZE=1024`), 이 한계가 C10K 문제를 만든 배경 |
| [05. epoll — 이벤트 기반 O(1) 처리](./io-models/05-epoll.md) | `epoll_create()`/`epoll_ctl()`/`epoll_wait()` 세 단계, 커널이 관심 fd 목록을 내부적으로 유지해 매번 전달할 필요가 없는 이유, 레벨 트리거(LT) vs 엣지 트리거(ET) 차이, **Redis의 `ae_epoll.c`가 `epoll_wait()`으로 이벤트를 처리하는 방식** |
| [06. I/O 모델이 백엔드에 미치는 영향](./io-models/06-io-model-backend-impact.md) | **Tomcat(스레드 풀, Blocking I/O) vs Netty/WebFlux(이벤트 루프, epoll) 처리 모델 비교**, 1000 동시 연결에서 각각의 스레드 수와 컨텍스트 스위칭 비용 차이, **Java 21 Virtual Thread가 OS 스레드 부족 문제를 우회하는 방식**, 어떤 워크로드에서 WebFlux가 유리한가 |

</details>

<br/>

### 🔹 Chapter 4: 파일 시스템과 디스크 I/O

> **핵심 질문:** 파일 하나를 읽을 때 커널에서 어떤 레이어를 거치는가? Sequential I/O가 Random I/O보다 빠른 커널 레벨 이유는? `fsync()`가 없으면 어떤 데이터가 사라지는가?

<details>
<summary><b>VFS부터 inode 구조까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. VFS — 리눅스가 파일 시스템을 추상화하는 방식](./filesystem-disk-io/01-vfs.md) | VFS(Virtual File System)가 ext4/XFS/tmpfs/procfs를 동일한 `open()`/`read()` 인터페이스로 다루는 레이어 구조, `inode`/`dentry`/`file` 오브젝트가 VFS에서 하는 역할, `/proc`과 `/sys`가 실제로 디스크가 없는 가상 파일 시스템인 이유 |
| [02. 디스크 I/O 스택 — 애플리케이션에서 디바이스까지](./filesystem-disk-io/02-disk-io-stack.md) | 애플리케이션 → VFS → 파일 시스템 → 블록 레이어(I/O 스케줄러) → 디바이스 드라이버 전 과정, 블록 레이어의 I/O 스케줄러(CFQ/deadline/noop)가 요청을 병합하고 정렬하는 방식, `iostat -xz 1`에서 각 수치가 어느 레이어를 반영하는지 |
| [03. Sequential vs Random I/O — HDD와 SSD의 진실](./filesystem-disk-io/03-sequential-vs-random-io.md) | HDD에서 Sequential I/O가 빠른 이유(헤드 이동 최소화), SSD에서도 Sequential이 유리한 이유(내부 병렬성, 쓰기 증폭), **MySQL InnoDB가 랜덤 쓰기를 순차 쓰기로 변환하는 Double Write Buffer**, **Kafka가 오직 Sequential Append로 설계된 이유** |
| [04. fsync와 내구성 — 커널 버퍼 플러시의 시점](./filesystem-disk-io/04-fsync-durability.md) | `write()` 후 데이터가 커널 Page Cache에 머물다 나중에 디스크에 반영되는 흐름, `fsync()`/`fdatasync()`가 강제 플러시하는 방식, **MySQL `innodb_flush_method`(fsync vs O_DIRECT vs O_DSYNC) 각 옵션의 실제 커널 동작**, 서버 재시작 시 데이터가 유실되는 시나리오와 예방 |
| [05. inode와 파일 시스템 구조 — 파일 삭제의 실체](./filesystem-disk-io/05-inode-filesystem.md) | inode가 파일 메타데이터(크기, 권한, 블록 포인터)를 저장하는 구조, 파일 삭제가 `unlink()`로 링크 카운트를 줄이는 것이며 `count=0`일 때만 공간이 해제되는 이유, 로그 파일 삭제 후 공간이 해제되지 않는 `lsof +L1` 시나리오와 해결 |

</details>

<br/>

### 🔹 Chapter 5: 네트워크 스택 — 커널에서 소켓까지

> **핵심 질문:** `send()` 호출 시 데이터가 실제로 전송되는 시점은 언제인가? Listen Backlog가 꽉 차면 어떤 일이 일어나는가? Zero-copy가 실제로 어떤 복사를 줄이는가?

<details>
<summary><b>소켓 버퍼부터 Zero-copy sendfile()까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 소켓과 커널 버퍼 — send/recv의 실체](./network-stack/01-socket-kernel-buffer.md) | `send()`가 실제로 하는 일은 커널 송신 버퍼(`sk_buff`)에 복사까지이며 실제 전송은 커널이 비동기로 하는 구조, 수신 버퍼가 가득 찰 때 TCP 흐름 제어(Zero Window)가 발동하는 과정, `ss -tmn`으로 소켓 버퍼 상태 확인 |
| [02. TCP 소켓 상태와 커널 — Accept Queue의 한계](./network-stack/02-tcp-socket-state.md) | SYN Backlog(3-way handshake 중인 연결)와 Accept Queue(완성된 연결)의 차이, `net.core.somaxconn`과 `tcp_max_syn_backlog` 설정이 각 큐의 크기를 결정하는 방식, 큐가 가득 찰 때 클라이언트에서 발생하는 `Connection refused` vs 무응답 차이 |
| [03. 소켓 옵션 — TCP_NODELAY부터 SO_REUSEPORT까지](./network-stack/03-socket-options.md) | `TCP_NODELAY`가 Nagle 알고리즘(작은 패킷 묶기)을 비활성화해 지연을 줄이는 방식, `SO_KEEPALIVE`로 커널이 유휴 연결을 주기적으로 검사하는 원리, `SO_REUSEPORT`로 여러 프로세스가 같은 포트를 나눠 받는 방식과 Nginx/Tomcat에서의 활용 |
| [04. 커널 네트워크 파라미터 튜닝 — 실전 설정](./network-stack/04-kernel-network-tuning.md) | `net.core.somaxconn`, `tcp_max_syn_backlog`, `net.ipv4.tcp_tw_reuse` 각 파라미터가 어떤 커널 동작을 제어하는지, `TIME_WAIT` 소켓이 대량으로 쌓이는 원인과 진단(`ss -s`), 고트래픽 서버에서 `ulimit -n` 파일 디스크립터 한도와의 관계 |
| [05. Zero-copy — sendfile()이 복사를 줄이는 방식](./network-stack/05-zero-copy-sendfile.md) | 일반 파일 전송의 4단계 복사(디스크→커널 버퍼→유저 버퍼→소켓 버퍼→NIC)와 `sendfile()`의 2단계(디스크→커널 버퍼→NIC) 비교, `strace`로 `sendfile()` 시스템 콜 확인, **Kafka Consumer가 `sendfile()`로 디스크 세그먼트를 직접 소켓에 전달하는 구조**, Nginx의 `sendfile on` 설정 |

</details>

<br/>

### 🔹 Chapter 6: cgroups와 namespace — 컨테이너의 실체

> **핵심 질문:** Docker 컨테이너는 가상 머신과 무엇이 다른가? `--memory=512m`이 커널에서 실제로 어떻게 동작하는가? 컨테이너 안의 JVM은 왜 OOM에 취약한가?

<details>
<summary><b>Linux namespace부터 seccomp까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Linux namespace — 컨테이너 격리의 구현](./cgroups-namespace/01-linux-namespace.md) | PID namespace(컨테이너 안에서 PID 1이 가능한 이유), Network namespace(독립된 네트워크 스택), Mount namespace(독립된 파일 시스템 뷰), UTS namespace(독립된 hostname), `lsns`로 실행 중인 namespace 목록 확인, `unshare` 명령어로 직접 namespace 생성 실험 |
| [02. cgroups v2 — 자원 제한의 실체](./cgroups-namespace/02-cgroups-v2.md) | **`docker run --memory=512m --cpus=1.5`가 `/sys/fs/cgroup/` 아래에 실제로 쓰는 값**, CPU CFS 쿼터(`cpu.cfs_quota_us`)로 CPU 사용률을 제한하는 방식, 메모리 한도 초과 시 커널이 페이지를 회수하거나 OOM Killer를 발동하는 순서, `systemd-cgls`로 cgroup 트리 확인 |
| [03. 컨테이너 네트워킹 — veth pair와 iptables](./cgroups-namespace/03-container-networking.md) | Docker `bridge` 네트워크가 `veth pair`(가상 이더넷 인터페이스 쌍)로 컨테이너와 호스트를 연결하는 방식, `iptables` NAT 규칙이 컨테이너 포트를 호스트 포트로 포워딩하는 과정, `ip link show`와 `iptables -t nat -L`로 실제 규칙 확인, 컨테이너 간 통신이 bridge를 거치는 경로 |
| [04. OOM in Containers — JVM과 cgroups의 충돌](./cgroups-namespace/04-oom-in-containers.md) | **컨테이너 메모리 한도 512MB + JVM `-Xmx1024m` 설정 시 OOM이 발생하는 경위**, Java 8이 컨테이너 메모리 한도를 무시하고 호스트 메모리를 기준으로 Heap을 계산하는 문제, **`-XX:+UseContainerSupport`(Java 10+)와 `-XX:MaxRAMPercentage=75.0`으로 해결하는 방법**, cgroups `memory.oom_control`으로 OOM Kill 동작 제어 |
| [05. seccomp와 capabilities — 컨테이너 보안의 구현](./cgroups-namespace/05-seccomp-capabilities.md) | Linux capabilities가 root 권한을 세분화하는 방식(`CAP_NET_BIND_SERVICE`, `CAP_SYS_PTRACE` 등), Docker 기본 컨테이너가 drop하는 capabilities 목록과 이유, seccomp 프로파일이 허용/차단할 시스템 콜을 필터링하는 방식, rootless 컨테이너가 가능한 원리 |

</details>

<br/>

### 🔹 Chapter 7: 성능 진단 — 도구와 방법론

> **핵심 질문:** `top`의 `wa`가 높을 때 어디를 먼저 봐야 하는가? 메모리 부족과 메모리 누수는 어떻게 구별하는가? `strace`와 `perf`로 애플리케이션 병목을 어떻게 찾는가?

<details>
<summary><b>CPU 분석부터 Flame Graph까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. CPU 분석 — us/sy/wa/id의 의미](./performance-diagnosis/01-cpu-analysis.md) | `top`/`htop`/`mpstat`에서 `us`(유저), `sy`(시스템 콜), `wa`(I/O 대기), `id`(유휴)가 각각 무엇을 의미하는지, `sy` 높음 = 시스템 콜 과다(확인 방법: `strace -c`), `wa` 높음 = 디스크 I/O 병목(확인 방법: `iostat`), Load Average가 CPU 수를 초과할 때의 의미 |
| [02. 메모리 분석 — cache/buffer의 실체와 누수 진단](./performance-diagnosis/02-memory-analysis.md) | `free -h`의 `available`이 `free + buff/cache`와 다른 이유, `vmstat`의 `si`/`so`(swap in/out)가 높을 때 의미하는 것, `/proc/meminfo`의 `Dirty`/`Writeback`/`AnonPages` 항목 해석, `valgrind`와 `/proc/<pid>/smaps`로 메모리 누수 위치를 좁히는 방법 |
| [03. I/O 분석 — await와 %util로 디스크 병목 찾기](./performance-diagnosis/03-io-analysis.md) | `iostat -xz 1`의 `await`(평균 I/O 대기 시간), `r_await`/`w_await`(읽기/쓰기 분리), `%util`(디스크 포화도) 의미와 SSD에서 `%util`이 100%여도 처리량이 남을 수 있는 이유, `iotop -bo`로 프로세스별 I/O 사용량 확인, MySQL/Redis I/O 병목 진단 시나리오 |
| [04. 네트워크 분석 — 소켓 상태와 TIME_WAIT 진단](./performance-diagnosis/04-network-analysis.md) | `ss -tanp`로 `ESTABLISHED`/`TIME_WAIT`/`CLOSE_WAIT` 상태별 소켓 수 확인, TIME_WAIT 대량 누적의 원인(짧은 수명의 연결 반복)과 `net.ipv4.tcp_tw_reuse` 설정으로 완화하는 방법, `sar -n DEV 1`으로 네트워크 처리량 측정, 수신 큐(`Recv-Q`) 누적이 의미하는 것 |
| [05. strace와 perf — 시스템 콜 추적과 Flame Graph](./performance-diagnosis/05-strace-perf.md) | `strace -c -p <pid>`로 프로세스의 시스템 콜 통계(호출 횟수, 소요 시간)를 수집하는 방법, `strace`로 애플리케이션이 예상보다 많은 `read()`/`write()`를 호출하는 패턴을 발견하는 실험, `perf record`로 CPU Flame Graph를 생성해 핫스팟 함수를 시각화하는 과정, Java 애플리케이션에 perf 적용 시 주의사항 |

</details>

---

## 🧪 실험 환경

모든 실험은 아래 Docker Compose 환경에서 재현 가능합니다.

```yaml
# docker-compose.yml
services:
  lab:
    image: ubuntu:22.04
    privileged: true          # 커널 기능 실험용 (namespace, cgroups)
    pid: host                 # 호스트 PID namespace 공유
    volumes:
      - /proc:/host-proc:ro   # 호스트 /proc 마운트
      - /sys:/host-sys:ro
    command: /bin/bash
    stdin_open: true
    tty: true

  resource-monitor:
    image: nicolargo/glances
    pid: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "61208:61208"
```

```bash
# 컨테이너 접속 후 실험 도구 설치
apt-get update && apt-get install -y \
  strace sysstat procps iproute2 \
  linux-tools-common linux-tools-generic \  # perf
  bpfcc-tools                               # eBPF 도구 (bpftrace 등)
```

---

## 🔬 공통 진단 명령어 세트

```bash
# ─────────────── 프로세스/스레드 ───────────────
strace -c -p <pid>              # 시스템 콜 통계 (호출 횟수 + 소요 시간)
strace -e trace=read,write,open,close -p <pid>  # I/O 시스템 콜만 추적
cat /proc/<pid>/status          # 프로세스 상태 (VmRSS, Threads 등)
cat /proc/<pid>/maps            # 가상 주소 공간 전체 매핑
ls -la /proc/<pid>/fd/          # 열린 파일 디스크립터 목록

# ─────────────── 메모리 ───────────────
cat /proc/meminfo               # 시스템 메모리 전체 현황
cat /proc/<pid>/smaps           # 프로세스 메모리 세그먼트별 세부 사용량
vmstat 1 10                     # 메모리/swap/I/O 1초 간격 10회
free -h                         # 메모리 사용량 요약 (buff/cache 포함)

# ─────────────── I/O ───────────────
iostat -xz 1                    # 디스크 I/O 통계 (await, %util 포함)
iotop -bo                       # 프로세스별 실시간 I/O 사용량

# ─────────────── 네트워크 소켓 ───────────────
ss -tanp                        # 모든 TCP 소켓 상태 + 프로세스 정보
ss -s                           # 소켓 통계 요약 (TIME_WAIT 수 등)
cat /proc/net/tcp               # TCP 연결 상태 raw 데이터

# ─────────────── cgroups (컨테이너 실험) ───────────────
cat /sys/fs/cgroup/memory/docker/<id>/memory.usage_in_bytes
cat /sys/fs/cgroup/memory/docker/<id>/memory.limit_in_bytes
cat /sys/fs/cgroup/cpu/docker/<id>/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/docker/<id>/cpu.cfs_period_us
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 실무에서 중요한가** | 실제 장애 / 성능 문제와 이 개념의 연결 |
| 😱 **흔한 실수** | Before — 커널을 모를 때의 접근과 그 결과 |
| ✨ **올바른 접근** | After — 커널을 알고 난 후의 설계/운영 |
| 🔬 **내부 동작 원리** | 커널 레벨 분석 + `/proc` 파일 시스템 + ASCII 구조도 |
| 💻 **실전 실험** | 재현 가능한 시나리오 + `strace` / `/proc` / 진단 명령어 세트 |
| 📊 **성능/비용 비교** | Blocking vs Non-Blocking, Page Cache 유무, fork() 비용 등 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "Redis BGSAVE 원리를 제대로 이해하고 싶다" — Redis 연계 집중 (3일)</b></summary>

<br/>

```
Day 1  Ch2-01  가상 메모리 → Page Table, TLB 기초
       Ch1-02  fork() + Copy-On-Write → Redis BGSAVE의 실체
Day 2  Ch2-03  Page Cache → Redis가 왜 메모리 DB인가
       Ch2-05  메모리 할당기 → jemalloc이 Redis에서 쓰이는 이유
Day 3  Ch2-06  OOM Killer → Redis 컨테이너 OOM 예방
       Ch6-04  OOM in Containers → JVM + 컨테이너 설정
```

</details>

<details>
<summary><b>🟡 "Spring WebFlux vs Tomcat 차이를 OS 수준으로 설명하고 싶다" — I/O 모델 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch3-01  파일 디스크립터와 시스템 콜 → 모든 I/O의 출발점
       Ch3-02  Blocking I/O → 프로세스가 잠드는 원리
Day 2  Ch3-03  Non-Blocking I/O → busy-wait의 문제
       Ch3-04  select/poll → O(N) 스캔의 한계
Day 3  Ch3-05  epoll → O(1) 이벤트 처리
Day 4  Ch3-06  I/O 모델 백엔드 영향 → Tomcat vs WebFlux 비교
       Ch1-04  컨텍스트 스위칭 → 스레드 수가 성능에 미치는 영향
Day 5  Ch1-03  스레드 모델 → Virtual Thread까지 연결
```

</details>

<details>
<summary><b>🔴 "커널 원리를 기저부터 전부 정복한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — 프로세스와 스레드
        → fork() CoW 실험, strace로 exec() 시스템 콜 추적, /proc/<pid>/maps 분석

2주차  Chapter 2 전체 — 메모리 관리
        → Page Fault 측정, Page Cache 효과 비교, OOM 시나리오 재현

3주차  Chapter 3 전체 — I/O 모델의 진화
        → select/poll/epoll 성능 비교, strace로 Redis epoll_wait() 확인

4주차  Chapter 4 전체 — 파일 시스템과 디스크 I/O
        → Sequential vs Random I/O fio 벤치마크, fsync 유무 데이터 내구성 실험

5주차  Chapter 5 전체 — 네트워크 스택
        → Accept Queue 포화 시뮬레이션, sendfile() vs 일반 전송 처리량 비교

6주차  Chapter 6 전체 — cgroups와 namespace
        → unshare로 namespace 직접 생성, cgroups 메모리 한도 + JVM OOM 재현

7주차  Chapter 7 전체 — 성능 진단
        → CPU Flame Graph 생성, strace로 시스템 콜 병목 탐색, 실제 장애 시나리오 진단
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 이 레포와 연결되는 챕터 |
|------|----------|----------------------|
| [redis-deep-dive](https://github.com/dev-book-lab/redis-deep-dive) | Redis 내부 아키텍처, 영속성, 캐싱 패턴 | Ch1-02(fork+CoW → BGSAVE), Ch2-05(jemalloc), Ch3-05(epoll → 단일 스레드 이벤트 루프) |
| [mysql-deep-dive](https://github.com/dev-book-lab/mysql-deep-dive) | InnoDB 내부 구조, MVCC, 인덱스 | Ch2-03(Page Cache → innodb_buffer_pool_size), Ch4-04(fsync → innodb_flush_method), Ch4-03(Sequential I/O → InnoDB) |
| [kafka-deep-dive](https://github.com/dev-book-lab/kafka-deep-dive) | 파티션, 복제, 컨슈머 그룹 | Ch2-03(Page Cache → Kafka 성능), Ch4-03(Sequential Append), Ch5-05(sendfile → Zero-copy) |
| [docker-deep-dive](https://github.com/dev-book-lab/docker-deep-dive) | 이미지, 컨테이너, 네트워크 | Ch6-01(namespace → 컨테이너 격리), Ch6-02(cgroups → 자원 제한), Ch6-03(veth + iptables → 컨테이너 네트워킹) |
| [network-deep-dive](https://github.com/dev-book-lab/network-deep-dive) | TCP/IP, HTTP, 로드 밸런싱 | Ch3-05(epoll → 고성능 서버), Ch5-01~05(소켓 커널 레벨 동작), Ch5-04(네트워크 파라미터 튜닝) |

> 💡 이 레포는 위 모든 레포의 **기저 원리**를 다룹니다. 각 레포에서 "왜?"라는 질문이 생길 때마다 여기로 돌아오세요.

---

## 🙏 Reference

- [Linux Kernel Development, 3rd Edition — Robert Love](https://www.amazon.com/Linux-Kernel-Development-Robert-Love/dp/0672329468)
- [The Linux Programming Interface — Michael Kerrisk](https://man7.org/tlpi/)
- [Systems Performance: Enterprise and the Cloud, 2nd Edition — Brendan Gregg](https://www.brendangregg.com/systems-performance-2nd-edition-book.html)
- [Understanding the Linux Kernel — Bovet & Cesati](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/)
- [Linux man pages](https://man7.org/linux/man-pages/)
- [Brendan Gregg's Blog](http://www.brendangregg.com/blog/) — 성능 분석 방법론의 바이블
- [Linux `/proc` 파일 시스템 문서](https://www.kernel.org/doc/html/latest/filesystems/proc.html)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"리눅스에 배포하는 것과, 커널이 I/O와 메모리를 어떻게 관리해 내 애플리케이션 성능을 결정하는지 아는 것은 다르다"*

</div>
