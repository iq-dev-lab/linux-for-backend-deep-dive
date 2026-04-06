# 01. Linux namespace — 컨테이너 격리의 구현

## 🎯 핵심 질문

- namespace는 무엇이고, 가상 머신과 어떻게 다른가?
- PID namespace에서 컨테이너 내부 PID 1이 가능한 원리는?
- Network namespace가 컨테이너에 독립된 네트워크 스택을 주는 방식은?
- `unshare`로 namespace를 직접 생성해 컨테이너 동작을 재현할 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Docker 없이 컨테이너를 직접 만들 수 있다.** namespace + cgroups + OverlayFS 세 가지를 조합하면 Docker가 하는 일을 그대로 재현할 수 있다. 이 원리를 알면 컨테이너 이상 동작을 커널 레벨에서 진단할 수 있다.

**`kubectl exec` 후 `ps aux`에서 호스트 프로세스가 보이지 않는 이유는?** PID namespace 덕분이다. 각 컨테이너는 독립된 PID 공간을 가지고, 컨테이너 내부에서는 자신의 프로세스만 볼 수 있다. 하지만 호스트에서는 컨테이너의 모든 프로세스가 보인다.

**컨테이너가 호스트 네트워크 스택을 공유하지 않아도 같은 포트 번호를 쓸 수 있는 이유는?** Network namespace가 각 컨테이너에 독립된 네트워크 스택(인터페이스, 라우팅 테이블, iptables 규칙, 소켓)을 제공한다. 컨테이너 A와 B가 각각 포트 8080을 listen해도 충돌하지 않는다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: 컨테이너를 VM처럼 생각
# "컨테이너는 완전히 격리된 OS다"
# → 아님. 커널을 공유한다!
$ docker run ubuntu uname -r
5.15.0-76-generic  ← 호스트 커널 버전과 동일
# 컨테이너는 격리된 뷰를 제공할 뿐, 커널 자체는 공유

# 실수 2: PID namespace 모르고 프로세스 관리
$ docker exec -it mycontainer ps aux
PID  CMD
  1  java -jar app.jar   ← 컨테이너 내부 PID 1
  50 ps aux

# "PID가 1이면 systemd처럼 동작하나?"
# → 아님. 컨테이너 PID namespace에서의 PID 1일 뿐
# → 호스트에서는 다른 PID를 가짐
$ ps aux | grep "java -jar"
1234  java -jar app.jar  ← 호스트에서의 실제 PID

# 실수 3: --net=host의 의미를 모름
$ docker run --net=host myapp  # Network namespace 공유
# → 컨테이너가 호스트 네트워크 스택 직접 사용
# → 포트 충돌 위험, 격리 없음
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 실행 중인 namespace 목록 확인
$ lsns
        NS TYPE   NPROCS    PID USER    COMMAND
4026531835 cgroup    120      1 root    /sbin/init
4026531836 ipc       120      1 root    /sbin/init
4026531837 mnt       115      1 root    /sbin/init
4026531838 net       115      1 root    /sbin/init
4026531839 pid       115      1 root    /sbin/init
4026532456 mnt         1  12345 root    /pause  ← Pod 인프라 컨테이너
4026532457 net         2  12345 root    /pause  ← 컨테이너와 공유
4026532458 pid         1  12346 app     java -jar app.jar

# 컨테이너의 namespace 확인
$ PID=$(docker inspect --format '{{.State.Pid}}' mycontainer)
$ ls -la /proc/$PID/ns/
lrwxrwxrwx ns/cgroup -> cgroup:[4026532459]
lrwxrwxrwx ns/ipc    -> ipc:[4026532460]
lrwxrwxrwx ns/mnt    -> mnt:[4026532461]
lrwxrwxrwx ns/net    -> net:[4026532462]
lrwxrwxrwx ns/pid    -> pid:[4026532463]
lrwxrwxrwx ns/uts    -> uts:[4026532464]

# 컨테이너 네트워크 namespace에 직접 진입
$ nsenter --target $PID --net -- ip addr show
# 컨테이너의 네트워크 인터페이스 확인 (격리된 스택)
```

---

## 🔬 내부 동작 원리

### namespace의 종류와 역할

```
리눅스 namespace (Linux 3.8+ 완성):

namespace   격리 대상               핵심 효과
──────────────────────────────────────────────────────
PID         프로세스 ID 공간         컨테이너 내 PID 1 가능
Network     네트워크 스택             독립된 NIC, IP, 라우팅
Mount       파일 시스템 마운트 트리   독립된 파일 시스템 뷰
UTS         hostname, domainname     독립된 hostname
IPC         System V IPC, POSIX MQ  독립된 프로세스간 통신
User        UID/GID 매핑             컨테이너 root = 호스트 비root
Cgroup      cgroup 루트 뷰           컨테이너에서 cgroup 정보 격리
Time        시계 오프셋              (Linux 5.6+, 드물게 사용)

VM과의 핵심 차이:
  VM:        하이퍼바이저가 하드웨어를 가상화, 게스트 OS 각각 실행
  Container: 커널 공유, namespace로 격리된 뷰만 제공
  → Container가 훨씬 가볍고 빠름 (시작 시간 ms vs 초)
  → Container는 커널 취약점이 공유됨 (VM은 격리)
```

### PID namespace 상세

```
호스트 PID 공간:
  PID 1: systemd (init)
  PID 100: sshd
  PID 500: dockerd
  PID 1234: [컨테이너 java 프로세스]  ← 호스트에서 보이는 PID

컨테이너 PID namespace:
  PID 1: java -jar app.jar  ← 컨테이너 내부 PID (호스트 PID 1234)
  PID 2: sh                  ← 컨테이너 내부 PID

  namespace 생성 원리:
    clone(CLONE_NEWPID, ...)
    → 새 PID namespace 생성
    → 첫 번째 프로세스 = PID 1 (새 namespace 기준)
    → 이 프로세스의 부모가 죽으면 PID 1도 사라짐

PID 1의 특별한 역할:
  - 고아 프로세스(zombie) 수거 (init 역할)
  - 시그널 핸들러 등록 않으면 SIGTERM 무시!
  → tini, dumb-init을 PID 1로 써야 하는 이유

확인 방법:
  $ docker exec container cat /proc/1/status | grep Pid
  Pid: 1          ← 컨테이너 namespace 기준
  NSpid: 1 1234   ← 1=컨테이너 namespace, 1234=호스트 PID
```

### Network namespace 상세

```
Network namespace 생성 시 독립된 구조:

  ┌─────────────────────────────────────────┐
  │ 컨테이너 Network namespace                │
  │  lo (loopback)        127.0.0.1         │
  │  eth0                 172.17.0.2/16     │ ← veth pair로 연결
  │  라우팅 테이블: default via 172.17.0.1     │
  │  iptables 규칙: 컨테이너 전용               │
  │  소켓: 독립된 포트 공간                      │
  └─────────────────────────────────────────┘
           │ veth pair
  ┌─────────────────────────────────────────┐
  │ 호스트 Network namespace                  │
  │  docker0 (bridge)     172.17.0.1/16     │
  │  eth0                 192.168.1.100     │
  │  veth12345678 (peer)  컨테이너 eth0과 연결  │
  └─────────────────────────────────────────┘

컨테이너 포트 바인딩 (-p 8080:80):
  컨테이너: eth0:80 listen
  iptables DNAT: 호스트:8080 → 컨테이너 IP:80
  → 서로 다른 Network namespace에서 같은 포트 번호 사용 가능
```

### Mount namespace 상세

```
Mount namespace = 마운트 포인트 트리의 독립된 뷰

호스트 마운트 트리:
  /         → /dev/sda1 (ext4)
  /proc     → proc
  /sys      → sysfs
  /home     → /dev/sdb1 (ext4)

컨테이너 Mount namespace:
  /         → overlayfs (레이어드 파일 시스템)
  /proc     → proc (컨테이너 PID namespace 기준)
  /sys      → sysfs
  /dev      → devtmpfs (제한된 장치)
  /tmp      → tmpfs

컨테이너 루트 파일 시스템 (OverlayFS):
  ┌─────────────────────────────────┐
  │ 컨테이너 레이어 (쓰기 가능)           │ ← 실행 중 변경사항
  ├─────────────────────────────────┤
  │ 이미지 레이어 3 (읽기 전용)           │
  ├─────────────────────────────────┤
  │ 이미지 레이어 2 (읽기 전용)           │
  ├─────────────────────────────────┤
  │ 이미지 레이어 1 (읽기 전용)           │ ← base image
  └─────────────────────────────────┘

  파일 접근: 위에서부터 찾아 내려옴 (CoW)
  수정 시: 하위 레이어에서 컨테이너 레이어로 Copy-Up
```

### unshare로 namespace 직접 생성

```bash
# PID + Mount namespace 생성 (미니 컨테이너)
$ unshare --pid --mount --fork --mount-proc bash

# 새 shell 안에서:
$ ps aux
PID CMD
  1 bash   ← 이 shell이 PID 1!
  2 ps aux

# 호스트 프로세스 보이지 않음 (PID namespace 격리)

# UTS namespace로 hostname 격리
$ unshare --uts bash
$ hostname mycontainer
$ hostname
mycontainer  ← 호스트에는 영향 없음

# Network namespace 생성
$ ip netns add myns
$ ip netns exec myns ip addr show
# lo만 있음 (격리된 빈 네트워크 스택)
$ ip link add veth0 type veth peer name veth1
$ ip link set veth1 netns myns
$ ip netns exec myns ip link set veth1 up
```

---

## 💻 실전 실험

### 실험 1: namespace 계층 구조 시각화

```bash
# 컨테이너 실행 후 namespace 비교
$ docker run -d --name ns-test nginx

# 호스트 namespace
$ ls -la /proc/1/ns/ | awk '{print $NF}'
net:[4026531838]  ← 호스트 net namespace

# 컨테이너 namespace
$ CPID=$(docker inspect --format '{{.State.Pid}}' ns-test)
$ ls -la /proc/$CPID/ns/ | awk '{print $NF}'
net:[4026532462]  ← 다른 번호! 독립된 net namespace

# namespace 유형별 비교
$ for ns in mnt uts ipc pid net; do
    host_ns=$(readlink /proc/1/ns/$ns)
    cont_ns=$(readlink /proc/$CPID/ns/$ns)
    echo "$ns: 호스트=$host_ns 컨테이너=$cont_ns"
    [ "$host_ns" = "$cont_ns" ] && echo "  → 공유됨!" || echo "  → 격리됨"
  done
```

### 실험 2: PID namespace 격리 확인

```bash
# 컨테이너 안에서 프로세스 목록
$ docker exec ns-test ps aux
PID USER CMD
  1 root  nginx: master process
  8 nginx nginx: worker process
# PID 1이 nginx! (컨테이너 namespace 기준)

# 호스트에서 실제 PID 확인
$ docker inspect --format '{{.State.Pid}}' ns-test
12345  ← 호스트에서의 실제 PID

# 호스트에서 컨테이너 프로세스 확인
$ ps aux | grep nginx
12345 root nginx: master process  ← 호스트에서는 다른 PID
12400 nginx nginx: worker process
```

### 실험 3: Network namespace 격리 실험

```bash
# 두 컨테이너가 같은 포트 8080을 내부적으로 사용
$ docker run -d -p 8081:8080 --name app1 myapp
$ docker run -d -p 8082:8080 --name app2 myapp
# 각 컨테이너의 Network namespace에서 8080 listen → 충돌 없음

# 컨테이너 네트워크 인터페이스 확인
$ docker exec app1 ip addr show
2: eth0: <BROADCAST,MULTICAST,UP> ...
    inet 172.17.0.2/16   ← 컨테이너 자체 IP

$ docker exec app2 ip addr show
2: eth0: <BROADCAST,MULTICAST,UP> ...
    inet 172.17.0.3/16   ← 다른 IP (다른 Network namespace)

# 호스트에서 veth pair 확인
$ ip link show | grep veth
veth12345678: <...> master docker0  ← app1용
veth87654321: <...> master docker0  ← app2용
```

---

## 📊 성능/비용 비교

| 격리 방식 | 시작 시간 | 메모리 오버헤드 | 커널 공유 | 격리 수준 |
|---------|---------|-------------|---------|---------|
| VM (KVM) | 수~수십 초 | 수백 MB (게스트 OS) | 아님 | 완전 격리 |
| Container | 수십~수백 ms | 수 MB | 공유 | 커널 레벨 격리 |
| Process | 수 ms | 수 KB | 공유 | 없음 |

**namespace 생성 비용**:

```
unshare 시스템 콜 비용:
  PID namespace:     ~수십 μs
  Network namespace: ~수백 μs (네트워크 스택 초기화)
  Mount namespace:   ~수십 μs
  User namespace:    ~수십 μs

Docker 컨테이너 시작 시간 분해:
  이미지 레이어 마운트 (OverlayFS): ~수십 ms
  namespace 생성:                 ~수 ms
  cgroups 설정:                   ~수 ms
  네트워크 설정 (veth, bridge):   ~수십 ms
  PID 1 프로세스 시작:             ~수십 ms
  합계:                           ~100~300ms
```

---

## ⚖️ 트레이드오프

**namespace 격리의 한계**

namespace는 커널 자원의 뷰를 격리하지만, 커널 자체를 격리하지 않는다. 컨테이너 내부의 커널 취약점 공격은 호스트 커널에도 영향을 줄 수 있다. VM은 각자 독립된 커널을 가지므로 이 공격 벡터가 없다. 보안이 매우 중요한 환경에서는 gVisor나 Kata Containers처럼 샌드박스 레이어를 추가한다.

**User namespace의 장단점**

User namespace를 사용하면 컨테이너 내부의 root가 호스트에서는 비root 사용자로 매핑된다. rootless 컨테이너가 가능해져 보안이 향상된다. 하지만 일부 시스템 콜이 제한되고, 특정 기능(예: CAP_NET_ADMIN)은 호스트에서 허용이 필요할 수 있다.

---

## 📌 핵심 정리

```
namespace = 커널 자원의 격리된 뷰
  컨테이너 = namespace + cgroups + OverlayFS 조합

주요 namespace:
  PID     → 독립된 프로세스 ID 공간 (컨테이너 내 PID 1)
  Network → 독립된 네트워크 스택 (NIC, IP, 라우팅, 소켓)
  Mount   → 독립된 파일 시스템 트리 (OverlayFS)
  UTS     → 독립된 hostname
  IPC     → 독립된 프로세스간 통신
  User    → UID/GID 매핑 (rootless 컨테이너)

VM vs Container:
  VM:  하이퍼바이저 + 게스트 OS (커널 격리)
  Container: namespace (커널 공유, 뷰만 격리)

핵심 진단 명령어:
  lsns                      → 실행 중인 namespace 목록
  ls -la /proc/<pid>/ns/    → 프로세스의 namespace 확인
  nsenter --target <pid> --net -- cmd  → namespace 진입
  unshare --pid --mount --fork bash   → 직접 namespace 생성
  ip netns list              → Network namespace 목록
```

---

## 🤔 생각해볼 문제

**Q1.** Kubernetes Pod 안에 여러 컨테이너가 있을 때, 이 컨테이너들이 공유하는 namespace와 독립적인 namespace는 무엇인가?

<details>
<summary>해설 보기</summary>

Kubernetes Pod는 **Network namespace를 공유**한다. Pod 내 모든 컨테이너는 같은 IP 주소, 같은 네트워크 인터페이스, 같은 localhost를 사용한다. 이것이 사이드카 패턴(앱 컨테이너 + 프록시 컨테이너)이 가능한 이유다. **PID namespace는 기본적으로 독립**이지만, `shareProcessNamespace: true`로 공유할 수 있다. **Mount namespace는 독립**이며, 볼륨 마운트로만 공유한다. 이 공유 구조를 구현하는 것이 각 Pod의 `/pause` (인프라 컨테이너)다. pause 컨테이너가 namespace를 먼저 생성하고, 다른 컨테이너들이 이 namespace에 join한다.

</details>

**Q2.** 컨테이너가 `/proc/sys/kernel/hostname`에 쓰기를 시도한다. UTS namespace가 있어도 호스트의 hostname이 바뀔 수 있는가?

<details>
<summary>해설 보기</summary>

**UTS namespace가 설정된 경우 바뀌지 않는다.** `/proc/sys/kernel/hostname`은 현재 UTS namespace에 바인딩되어 있다. 컨테이너가 이 파일에 쓰면 컨테이너의 UTS namespace에만 반영된다. 단, `--uts=host` 옵션으로 컨테이너가 호스트 UTS namespace를 공유하면 hostname 변경이 호스트에 영향을 준다. 보안 정책으로 `CAP_SYS_ADMIN` capability를 drop하면 hostname 변경을 막을 수 있다.

</details>

**Q3.** `docker exec -it container bash`로 컨테이너에 접속했을 때, bash 프로세스가 컨테이너의 namespace를 사용하는 원리는?

<details>
<summary>해설 보기</summary>

`docker exec`는 내부적으로 `setns()` 시스템 콜을 사용한다. 컨테이너의 기존 프로세스(PID 1 등)의 `/proc/<pid>/ns/` 디렉토리에서 namespace fd를 열고, `setns(ns_fd, nstype)`로 해당 namespace에 합류한다. 새로운 bash 프로세스는 기존 컨테이너의 PID/Network/Mount/UTS namespace에 진입해 컨테이너 내부와 동일한 환경으로 실행된다. 이것이 `nsenter` 명령어가 하는 일이기도 하다: `nsenter --target <pid> --all -- bash`.

</details>

---

**[홈으로 🏠](../README.md)** | **[다음: cgroups v2 ➡️](./02-cgroups-v2.md)**
