# 05. seccomp와 capabilities — 컨테이너 보안의 구현

## 🎯 핵심 질문

- Linux capabilities는 root 권한을 어떻게 세분화하는가?
- Docker 기본 컨테이너가 drop하는 capabilities는 무엇이고 왜인가?
- seccomp 프로파일이 시스템 콜을 필터링하는 방식은?
- rootless 컨테이너가 가능한 커널 레벨 원리는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**컨테이너 내부에서 `strace`가 안 된다.** 기본 Docker 컨테이너에서는 `CAP_SYS_PTRACE` capability가 없기 때문이다. `docker run --cap-add SYS_PTRACE`로 추가하면 사용할 수 있다. capabilities를 이해하면 "권한이 없어요" 오류의 정확한 원인을 진단할 수 있다.

**컨테이너가 80번 포트에 bind하려면 root 권한이 필요하다는 말이 있다.** 실제로 필요한 것은 `CAP_NET_BIND_SERVICE` capability이고, root일 필요는 없다. 컨테이너 내 비root 사용자도 이 capability가 있으면 1024 미만 포트에 바인딩 가능하다.

**Kubernetes Pod Security Standard의 `Restricted` 정책이 많은 것을 차단한다.** seccomp 프로파일 강제, 비root 실행, privilege escalation 금지 등이 포함된다. 이 정책이 무엇을 제한하는지 알아야 애플리케이션을 올바르게 설정할 수 있다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: 필요한 capability 파악 없이 --privileged 남용
$ docker run --privileged myapp  # 모든 capabilities 허용!
# → 컨테이너가 호스트 커널에 거의 제한 없이 접근
# → 컨테이너 탈출(escape) 공격 위험 급증
# → 대신 필요한 capability만 추가:
$ docker run --cap-add NET_ADMIN --cap-add SYS_PTRACE myapp

# 실수 2: seccomp=unconfined으로 보안 우회
$ docker run --security-opt seccomp=unconfined myapp
# → 모든 시스템 콜 허용 → 커널 취약점 공격 노출

# 실수 3: 컨테이너를 root로 실행
# Dockerfile:
# USER root  ← 또는 USER 명시 없음
# → 컨테이너 프로세스가 UID 0으로 실행
# → 컨테이너 탈출 시 호스트에서도 root
# 올바른 방법:
# RUN useradd -r -u 1000 appuser
# USER 1000
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 현재 프로세스의 capabilities 확인
$ cat /proc/$$/status | grep Cap
CapInh: 0000000000000000  ← 상속 가능한 capability
CapPrm: 000001ffffffffff  ← 허용된 capability
CapEff: 000001ffffffffff  ← 현재 유효한 capability
CapBnd: 000001ffffffffff  ← bounding set
CapAmb: 0000000000000000  ← ambient capability

# capsh로 디코딩
$ capsh --decode=000001ffffffffff
0x000001ffffffffff=cap_chown,cap_dac_override,...,cap_net_bind_service,...

# 컨테이너의 capabilities 확인
$ docker run --rm ubuntu cat /proc/1/status | grep Cap
CapInh: 00000000a80425fb  ← 기본 Docker 허용 목록 (제한됨)

# 특정 capability만 추가
$ docker run --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --cap-add SETUID --cap-add SETGID nginx

# Kubernetes Pod Security Context
cat << 'EOF'
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
      readOnlyRootFilesystem: true
EOF
```

---

## 🔬 내부 동작 원리

### Linux capabilities 상세

```
전통적 Unix 권한 모델의 문제:
  프로세스 = root (UID 0) → 모든 권한
  프로세스 = 비root → 제한된 권한
  → 한 가지 권한만 필요해도 root 전체를 줘야 함

Linux capabilities (Linux 2.2+):
  root 권한을 64개의 독립적인 capability로 분해
  각 capability는 개별적으로 부여/회수 가능

주요 capabilities:
  CAP_CHOWN          파일 소유자 변경
  CAP_DAC_OVERRIDE   파일 권한 무시 읽기/쓰기
  CAP_NET_BIND_SERVICE 1024 미만 포트 바인딩
  CAP_NET_ADMIN      네트워크 인터페이스/라우팅 설정
  CAP_SYS_ADMIN      마운트, ioctl, 기타 시스템 관리 (사실상 슈퍼 capability)
  CAP_SYS_PTRACE     다른 프로세스 추적 (strace, gdb)
  CAP_SYS_CHROOT     chroot() 호출
  CAP_KILL           임의 프로세스에 시그널 전송
  CAP_SETUID         UID 변경 (su, sudo)
  CAP_SETGID         GID 변경
  CAP_AUDIT_WRITE    커널 감사 로그 작성
  CAP_MKNOD          장치 파일 생성

Capability 집합:
  Permitted:   프로세스가 가질 수 있는 최대 capability
  Effective:   현재 커널이 실제로 확인하는 capability
  Inheritable: exec() 후 자식에게 전달될 수 있는 것
  Bounding:    Permitted를 제한하는 상한선
```

### Docker 기본 capability 설정

```
Docker 기본 허용 capability 목록 (drop됨을 표시):
  DROP: CAP_AUDIT_CONTROL    ← 감사 서브시스템 제어
  DROP: CAP_BLOCK_SUSPEND    ← 시스템 절전 차단
  DROP: CAP_DAC_READ_SEARCH  ← 파일 권한 무시 읽기
  DROP: CAP_IPC_LOCK         ← 메모리 잠금
  DROP: CAP_IPC_OWNER        ← IPC 객체 소유권 우회
  DROP: CAP_LINUX_IMMUTABLE  ← 불변 파일 속성 설정
  DROP: CAP_MAC_ADMIN        ← MAC 정책 변경
  DROP: CAP_MAC_OVERRIDE     ← MAC 정책 우회
  DROP: CAP_MKNOD            ← 장치 파일 생성
  DROP: CAP_NET_ADMIN        ← 네트워크 관리
  DROP: CAP_NET_BROADCAST    ← 브로드캐스트/멀티캐스트
  DROP: CAP_SYS_ADMIN        ← 시스템 관리 (마운트 등)
  DROP: CAP_SYS_BOOT         ← 재시작
  DROP: CAP_SYS_MODULE       ← 커널 모듈 로드/언로드
  DROP: CAP_SYS_NICE         ← 프로세스 우선순위 변경
  DROP: CAP_SYS_PACCT        ← 프로세스 계정
  DROP: CAP_SYS_PTRACE       ← ptrace (strace, gdb)
  DROP: CAP_SYS_RAWIO        ← 직접 I/O 포트 접근
  DROP: CAP_SYS_RESOURCE     ← 자원 한도 변경
  DROP: CAP_SYS_TIME         ← 시스템 시간 변경
  DROP: CAP_SYS_TTY_CONFIG   ← TTY 설정
  DROP: CAP_WAKE_ALARM       ← 알람으로 시스템 깨우기
  DROP: CAP_SYSLOG           ← 커널 로그 접근

기본 유지 (허용):
  CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FOWNER,
  CAP_FSETID, CAP_KILL, CAP_SETGID, CAP_SETUID,
  CAP_SETPCAP, CAP_NET_BIND_SERVICE, CAP_NET_RAW,
  CAP_SYS_CHROOT, CAP_MKNOD*, CAP_AUDIT_WRITE,
  CAP_SETFCAP
```

### seccomp (Secure Computing Mode)

```
seccomp = 프로세스가 사용할 수 있는 시스템 콜 필터

동작 방식:
  BPF(Berkeley Packet Filter) 프로그램으로 시스템 콜 규칙 정의
  각 시스템 콜 호출 시 BPF 프로그램이 평가:
    ALLOW:  시스템 콜 허용
    ERRNO:  오류 반환 (EPERM 등)
    KILL:   프로세스 즉시 종료
    TRAP:   SIGSYS 시그널 전송

Docker 기본 seccomp 프로파일:
  ~300개 시스템 콜 허용 (총 400여 개 중)
  차단되는 주요 시스템 콜:
    keyctl, add_key, request_key  ← 커널 키링 접근
    ptrace                        ← 프로세스 추적
    personality                   ← 실행 도메인 변경
    modify_ldt                    ← 세그먼트 테이블 수정
    kexec_load                    ← 새 커널 로드
    open_by_handle_at             ← inode 기반 파일 접근 (컨테이너 탈출)
    perf_event_open               ← 성능 카운터 (CPU 사이드채널)
    clone(new namespaces)         ← namespace 생성 제한

커스텀 seccomp 프로파일 예시:
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat",
                "mmap", "mprotect", "munmap", "brk",
                "socket", "connect", "accept", "send", "recv",
                "exit_group", "futex", "clock_gettime"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### rootless 컨테이너의 원리

```
전통적 컨테이너 (root 필요):
  namespace 생성: 일부 CLONE_NEW* 플래그는 CAP_SYS_ADMIN 필요
  cgroups v1 설정: /sys/fs/cgroup/ 쓰기 권한 필요 (root)
  veth 생성: CAP_NET_ADMIN 필요

rootless 컨테이너 (비root 실행, Linux 3.8+ User namespace):
  User namespace:
    컨테이너 내부 UID 0 → 호스트 UID 1000으로 매핑
    /proc/<pid>/uid_map: "0 1000 1"

  이것이 가능하게 하는 커널 기능:
    ① User namespace를 생성하면 그 안에서 비root도 namespace 생성 가능
    ② cgroups v2: 비root 사용자도 자신의 cgroup 하위에 생성 가능
    ③ UID 매핑으로 컨테이너 내 파일 소유권 처리

  구현:
    podman: User namespace 기반 rootless
    Docker rootless: rootlesskit + slirp4netns
    Kubernetes: rootless container runtime (containerd 1.7+)

  제약:
    - 일부 네트워크 기능 제한 (포트 1024 미만 불가 등)
    - 성능 약간 저하
    - 일부 volume mount 제약
```

---

## 💻 실전 실험

### 실험 1: capabilities 실험

```bash
# 비root 프로세스로 포트 80 바인딩 테스트
$ docker run --rm ubuntu python3 -c "
import socket
s = socket.socket()
s.bind(('', 80))
print('포트 80 바인딩 성공')
" 2>&1
# Permission denied: [Errno 13] → CAP_NET_BIND_SERVICE 없음

# CAP_NET_BIND_SERVICE 추가
$ docker run --rm --cap-drop ALL --cap-add NET_BIND_SERVICE \
  ubuntu python3 -c "
import socket
s = socket.socket()
s.bind(('', 80))
print('포트 80 바인딩 성공')
"
# 성공!

# 컨테이너 capabilities 비교
$ docker run --rm ubuntu capsh --print
Current: = cap_chown,cap_dac_override,...  ← 기본 허용 목록

$ docker run --rm --privileged ubuntu capsh --print
Current: = cap_chown,cap_dac_override,...,cap_sys_admin,...  ← 전부!

# strace 실행 (CAP_SYS_PTRACE 필요)
$ docker run --rm ubuntu strace ls 2>&1 | head -3
strace: attach: ptrace(PTRACE_SEIZE, 1): Operation not permitted

$ docker run --rm --cap-add SYS_PTRACE ubuntu strace ls 2>&1 | head -3
execve("/bin/ls", ["ls"], ...) = 0
```

### 실험 2: seccomp 프로파일 테스트

```bash
# 커스텀 seccomp 프로파일 작성 (chmod 차단)
$ cat > /tmp/no-chmod.json << 'EOF'
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["chmod", "fchmod", "fchmodat"],
      "action": "SCMP_ACT_ERRNO",
      "errnoRet": 1
    }
  ]
}
EOF

$ docker run --rm \
  --security-opt seccomp=/tmp/no-chmod.json \
  ubuntu bash -c "chmod 777 /tmp/test && echo 'chmod 성공'"
chmod: changing permissions of '/tmp/test': Operation not permitted
# seccomp이 chmod 시스템 콜을 차단

# 기본 seccomp(ptrace 차단) 확인
$ docker run --rm ubuntu strace ls
strace: attach: ptrace(PTRACE_SEIZE, 1): Operation not permitted

$ docker run --rm --security-opt seccomp=unconfined ubuntu strace ls 2>&1 | head -3
execve("/bin/ls", ["ls"], ...) = 0  ← 성공!
```

### 실험 3: rootless 컨테이너 UID 매핑 확인

```bash
# Docker rootless 모드 실행 (rootless Docker 설치 필요)
# 또는 User namespace로 직접 확인

# User namespace에서 UID 매핑
$ unshare --user --map-root-user bash
$ id
uid=0(root) gid=0(root)  ← namespace 안에서 root!
$ cat /proc/$$/uid_map
         0       1000          1  ← 내부 0 = 외부 1000

# 파일 생성
$ touch /tmp/ns_test_file
$ ls -la /tmp/ns_test_file
-rw-r--r-- 1 root root 0 ...  ← namespace 내부에서는 root 소유

# 다른 터미널(호스트)에서:
$ ls -la /tmp/ns_test_file
-rw-r--r-- 1 user1000 user1000 0 ...  ← 호스트에서는 UID 1000 소유
```

---

## 📊 성능/비용 비교

**보안 제한 레벨별 비교**:

```
레벨              설정                         보안   기능
최소 보안         --privileged                  ★     ★★★★★
낮은 보안         기본 Docker                   ★★★   ★★★★
높은 보안         --cap-drop ALL + 필요한 것만  ★★★★  ★★★
최고 보안         + seccomp custom              ★★★★★ ★★
```

**seccomp 오버헤드**:

```
seccomp 없음:       시스템 콜 기준선
seccomp (BPF):      ~0.5~5% 오버헤드 (시스템 콜 집약도에 따라)
                    CPU 바운드 작업: 오버헤드 거의 없음
                    I/O 바운드 작업: 약간 증가 (시스템 콜 많음)
```

---

## ⚖️ 트레이드오프

**최소 권한 원칙의 복잡성**

`--cap-drop ALL --cap-add <필요한것만>` 방식은 보안이 가장 좋지만, 애플리케이션이 실제로 어떤 capability가 필요한지 파악해야 한다. `strace`로 시스템 콜을 추적하거나, 애플리케이션 문서를 확인해야 한다. 잘못 drop하면 애플리케이션이 동작하지 않는다. 실용적으로는 기본 Docker 설정에서 시작해 필요한 것을 추가하는 방식이 더 안전하다.

**seccomp 커스텀 프로파일의 유지보수 부담**

커스텀 seccomp 프로파일은 애플리케이션 업데이트 시 새 시스템 콜이 추가될 수 있어 유지보수가 필요하다. Docker의 기본 seccomp 프로파일(`docker/default`)은 대부분의 애플리케이션에 충분하며, 커널 업그레이드에 따라 관리된다. `RuntimeDefault`(기본 프로파일)를 사용하고, 추가 차단이 필요한 경우에만 커스텀을 적용한다.

---

## 📌 핵심 정리

```
Linux capabilities:
  root 권한 = 64개의 독립 capability
  컨테이너: 필요한 것만 허용 (최소 권한 원칙)

Docker 기본: ~14개 capability drop
  SYS_ADMIN, NET_ADMIN, SYS_PTRACE 등 위험한 것 제거

자주 필요한 추가:
  --cap-add SYS_PTRACE    → strace, gdb
  --cap-add NET_ADMIN     → iptables, tcpdump
  --cap-add SYS_ADMIN     → mount, 기타 (위험!)

seccomp:
  시스템 콜 레벨 필터링 (BPF 기반)
  기본: ~300개 허용, 위험한 ~100개 차단
  커스텀 프로파일: json으로 정의

rootless 컨테이너:
  User namespace + UID 매핑으로 구현
  컨테이너 내 root → 호스트 비root (UID 1000 등)
  /proc/<pid>/uid_map으로 매핑 확인

Kubernetes 보안 설정:
  securityContext.runAsNonRoot: true
  capabilities.drop: ["ALL"]
  allowPrivilegeEscalation: false
  seccompProfile.type: RuntimeDefault

진단:
  cat /proc/<pid>/status | grep Cap → 현재 capability
  capsh --decode=<hex>              → 디코딩
  docker run --cap-add              → capability 추가
  --security-opt seccomp=<file>     → seccomp 프로파일 지정
```

---

## 🤔 생각해볼 문제

**Q1.** 컨테이너 내부에서 `ping` 명령어가 동작하지 않는다. 어떤 capability가 필요하고, 대안은 무엇인가?

<details>
<summary>해설 보기</summary>

`ping`은 raw socket을 사용하기 위해 `CAP_NET_RAW` capability가 필요하다. Docker 기본 설정에서 `CAP_NET_RAW`는 허용되어 있어 보통 동작한다. 만약 `--cap-drop ALL`로 모든 capability를 drop했다면 `--cap-add NET_RAW`를 추가해야 한다. 대안으로는 root 없이 동작하는 `ping`(setuid bit 활용), 또는 TCP/UDP 기반 연결 테스트(`curl`, `nc`)를 사용할 수 있다. Kubernetes에서 `privileged: false`이고 capability를 drop한 경우 `ping` 사용이 불가능할 수 있다.

</details>

**Q2.** `CAP_SYS_ADMIN`이 "슈퍼 capability"라고 불리는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

`CAP_SYS_ADMIN`은 매우 광범위한 권한을 포함한다. mount/umount, ioctl(다양한 장치 제어), 커널 키링 관리, cgroup 관리, 네임스페이스 생성, 프로세스 계정 등이 모두 이 하나의 capability로 제어된다. 실질적으로 `CAP_SYS_ADMIN`을 부여하면 컨테이너가 호스트 시스템에 대한 광범위한 권한을 얻는다. 컨테이너 탈출(escape) 취약점의 많은 부분이 `CAP_SYS_ADMIN`을 통해 이루어진다. Docker가 이것을 기본 drop 목록에 포함한 이유다.

</details>

**Q3.** Kubernetes `PodSecurityAdmission`의 `Restricted` 정책이 적용된 namespace에서 Redis를 실행하려면 어떤 설정이 필요한가?

<details>
<summary>해설 보기</summary>

`Restricted` 정책의 주요 요구사항을 만족해야 한다. `runAsNonRoot: true` — Redis 기본 이미지는 root로 실행되므로 `redis` 사용자(UID 999)로 실행하도록 설정해야 한다. `allowPrivilegeEscalation: false` — Redis는 이것을 필요로 하지 않으므로 설정 가능. `capabilities.drop: ["ALL"]` — Redis는 특별한 capability가 필요 없다. `seccompProfile.type: RuntimeDefault` — 기본 seccomp 프로파일 적용. `readOnlyRootFilesystem: true` — Redis는 `/data` 디렉토리에 쓰기가 필요하므로 볼륨 마운트 필수. 이런 설정으로 Redis는 Restricted 정책에서도 동작할 수 있다.

</details>

---

**[⬅️ 이전: OOM in Containers](./04-oom-in-containers.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — CPU 분석 ➡️](../performance-diagnosis/01-cpu-analysis.md)**
