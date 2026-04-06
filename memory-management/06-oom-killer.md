# 06. OOM Killer — 프로세스 종료 기준과 예방

## 🎯 핵심 질문

- 물리 메모리가 고갈되면 커널은 어떤 기준으로 종료할 프로세스를 선택하는가?
- `oom_score`와 `oom_score_adj`는 어떻게 계산되고 조정하는가?
- 컨테이너 환경에서 OOM은 어떻게 다르게 동작하는가?
- JVM OOM과 커널 OOM은 어떻게 구별하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Kubernetes Pod가 갑자기 재시작되는데 로그에 아무것도 없다.** 커널 OOM Killer가 JVM 프로세스를 종료하면 JVM 자체가 로그를 남길 기회가 없다. `dmesg | grep -i oom`으로 커널 로그를 확인해야 한다. 이 차이를 모르면 애플리케이션 버그를 찾다가 시간을 낭비한다.

**JVM이 `-Xmx4g` 제한을 지키는데도 컨테이너가 OOM으로 죽는다.** JVM Heap 외에도 Metaspace, Thread Stack, Direct Buffer, JIT 코드 캐시가 Native Memory를 추가로 사용한다. 컨테이너 메모리 limit을 `-Xmx`만 기준으로 설정하면 필연적으로 OOM이 발생한다.

**Redis 서버가 예고 없이 종료된다.** 메모리 사용량이 `maxmemory`를 넘지 않았는데도 종료될 수 있다. 커널의 OOM Killer가 Redis를 타겟으로 잡은 것이다. `oom_score_adj`로 Redis를 보호하거나, 다른 낮은 우선순위 프로세스가 먼저 죽도록 설정해야 한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: JVM OOM과 커널 OOM 혼동
# 애플리케이션 로그:
java.lang.OutOfMemoryError: Java heap space
   at java.util.Arrays.copyOf(Arrays.java:3210)
# → JVM Heap 부족. Heap 늘리면 해결

# 커널 OOM 시 로그 없음 (프로세스 즉시 종료)
# → Kubernetes: Pod CrashLoopBackOff, 이유 불명
# → dmesg 확인 필요!

# 실수 2: -Xmx만 보고 컨테이너 limit 설정
$ docker run --memory=4g java-app -Xmx4g  # ← 틀린 설정!
# JVM 전체 메모리 = Heap(4g) + Metaspace + Stack + Native
# 컨테이너 limit과 동일하게 설정 → OOM 확실

# 실수 3: oom_score_adj를 모른 채 중요한 서비스 방치
$ cat /proc/$(pgrep redis-server)/oom_score
700  ← 높은 점수 = OOM 타겟 우선순위 높음!
# Redis가 가장 먼저 죽을 수 있는 상황
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# OOM 발생 여부 및 원인 확인
$ dmesg | grep -i "oom\|killed process" | tail -20
[12345.678] Out of memory: Kill process 1234 (java) score 892 or sacrifice child
[12345.679] Killed process 1234 (java) total-vm:8388608kB, anon-rss:4194304kB

# oom_score 모니터링 (점수 높을수록 먼저 종료)
$ for pid in $(ls /proc | grep -E '^[0-9]+$'); do
    score=$(cat /proc/$pid/oom_score 2>/dev/null)
    comm=$(cat /proc/$pid/comm 2>/dev/null)
    [ "${score:-0}" -gt 100 ] && echo "score=$score pid=$pid $comm"
  done | sort -rn | head -10

# 중요 서비스 OOM 보호
$ echo -1000 > /proc/$(pgrep redis-server)/oom_score_adj
# -1000: 절대 OOM 종료 안 됨 (최고 보호)
# -999: 거의 종료 안 됨
# 0: 기본값
# +1000: 가장 먼저 종료

# JVM 컨테이너 OOM 예방
$ docker run --memory=6g java-app \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0
# 컨테이너 메모리(6GB)의 75% = 4.5GB Heap
# 나머지 1.5GB = Metaspace + Stack + Native
```

---

## 🔬 내부 동작 원리

### OOM 발생 조건

```
물리 메모리 고갈 과정:

1. 애플리케이션 메모리 할당 요청 (malloc, mmap 등)
2. 커널: 여유 물리 페이지 있는지 확인
3. 없음 → 페이지 회수 시도 (Reclaim):
   ① Page Cache Eviction (파일 캐시 해제)
   ② 스왑 아웃 (Swap 있는 경우)
   ③ slab 캐시 해제 (커널 내부 캐시)
4. 회수 후에도 부족 → OOM Killer 발동
   → oom_kill_process() 호출

vm.overcommit_memory 설정이 OOM 시점에 영향:
  0 (기본): 휴리스틱으로 overcommit 허용
  1: 항상 overcommit 허용 (OOM 가능성 높음)
  2: strict 제한 (overcommit 불허, malloc 실패 반환)
```

### oom_score 계산 방식

```
oom_score = 해당 프로세스의 종료 우선순위 (0~1000)
높을수록 먼저 종료

기본 계산 (커널 소스: oom_score_of_task()):
  기본점수 = (프로세스 RSS + swap 사용량) / 전체 물리 메모리 * 1000
  → 메모리를 많이 쓸수록 점수 높음

보정 요소:
  + root 프로세스: 약간 낮게 보정 (다만 안전장치 아님)
  + oom_score_adj (-1000~+1000): 수동 조정값 더해짐

최종:
  oom_score = badness(RSS기반 점수) + oom_score_adj

예시:
  Redis (RSS 6GB, 전체 8GB):  기본점수 750 + adj 0 = 750
  Nginx (RSS 200MB, 전체 8GB): 기본점수 25 + adj 0 = 25
  → Redis가 먼저 종료됨!

  adj 설정 후:
  Redis oom_score_adj = -900: 750 - 900 = -150 → 보호됨
  Nginx oom_score_adj = +500: 25 + 500 = 525 → 더 먼저 죽음
```

### 컨테이너 환경의 OOM

```
Docker/Kubernetes 메모리 제한 동작:

docker run --memory=512m app

커널 cgroups 설정:
  /sys/fs/cgroup/memory/docker/<id>/memory.limit_in_bytes = 536870912

컨테이너 내 프로세스가 한도 초과 시:
  경우 1: memory.oom_kill_disable = 0 (기본)
    → 컨테이너 내 OOM Killer 발동
    → 한도 초과 원인 프로세스 종료
    → 컨테이너는 계속 실행 (프로세스만 종료)

  경우 2: PID 1이 종료됨
    → Docker/Kubernetes가 컨테이너 재시작
    → Kubernetes: OOMKilled 상태로 기록

$ kubectl describe pod <pod-name>
  Last State: Terminated
    Reason: OOMKilled   ← 커널 OOM으로 종료된 증거
    Exit Code: 137      ← 128 + SIGKILL(9) = 137

dmesg에서 확인:
[  1234.567] Task in /kubepods/pod.../container... killed as a result of limit of /kubepods/pod...
[  1234.568] Memory cgroup out of memory: Kill process 5678 (java) score 1000 or sacrifice child
```

### JVM OOM vs 커널 OOM 구별

```
JVM OOM (java.lang.OutOfMemoryError):
  발생 위치: JVM 내부 (Heap, Metaspace, GC overhead 등)
  로그: Java 스택 트레이스 남음
  종료: JVM이 스스로 OOMError 던짐 → JVM 종료
  종료 코드: 1
  확인: 애플리케이션 로그, GC 로그

커널 OOM (OOM Killer):
  발생 위치: 커널 메모리 할당 실패 시
  로그: 없음 (프로세스 즉시 SIGKILL)
  종료: 커널이 SIGKILL 전송 (핸들러 없음)
  종료 코드: 137 (128 + 9)
  확인: dmesg, /var/log/syslog, Kubernetes Events

구별 방법:
  종료 코드 137 → 커널 OOM (SIGKILL)
  종료 코드 1   → JVM OOM (예외)

  $ dmesg -T | grep -i "killed process\|oom"
  [Thu Jan  1 00:00:00 2024] Killed process 1234 (java) ...
  ↑ 있으면 커널 OOM, 없으면 JVM OOM
```

### JVM 컨테이너 메모리 계획

```
컨테이너 limit = 4GB 설정 시:

잘못된 설정:
  -Xmx4g  ← Heap이 컨테이너 전체를 차지하려 함
  실제: Heap(4GB) + Metaspace(~256MB) + Thread Stack(~200MB) + ...
  → 컨테이너 한도 초과 → OOM Kill

올바른 설정:
  -XX:+UseContainerSupport           ← Java 10+ 컨테이너 인식
  -XX:MaxRAMPercentage=75.0          ← 컨테이너 메모리의 75%를 Heap으로
  (4GB * 75% = 3GB Heap)
  나머지 1GB = Metaspace + Stack + Native

또는 명시적:
  -Xmx3g                             ← Heap 명시
  -XX:MaxMetaspaceSize=256m          ← Metaspace 제한
  -Xss512k                           ← 스레드 스택 축소
  (-Xss512k * 200 threads = 100MB)

메모리 계획 공식:
  컨테이너 limit = Heap + Metaspace + (스레드 수 * 스택 크기) + Off-heap + 여유
  최소 여유: 15~20%
```

---

## 💻 실전 실험

### 실험 1: OOM Killer 동작 재현

```bash
# 주의: 실험용 격리 환경에서만 실행
$ docker run -it --memory=256m ubuntu:22.04 bash

# 컨테이너 안에서 메모리 고갈 테스트
$ apt-get install -y stress-ng
$ stress-ng --vm 1 --vm-bytes 300M --timeout 10s
# → 256MB 제한 초과
# → OOM Killer 발동
# → stress-ng 프로세스 종료

# 호스트에서 dmesg 확인
$ dmesg | tail -10 | grep -i oom
```

### 실험 2: oom_score 모니터링 스크립트

```bash
# 전체 프로세스 OOM 점수 순위
$ cat << 'EOF' > /tmp/oom_monitor.sh
#!/bin/bash
echo "=== OOM Score 순위 (높을수록 먼저 죽음) ==="
echo "Score  ADJ   PID    Name"
echo "-----  ----  -----  ----"
for pid in $(ls /proc | grep -E '^[0-9]+$'); do
    score=$(cat /proc/$pid/oom_score 2>/dev/null) || continue
    adj=$(cat /proc/$pid/oom_score_adj 2>/dev/null) || continue
    comm=$(cat /proc/$pid/comm 2>/dev/null) || continue
    rss=$(awk '/VmRSS/{print $2}' /proc/$pid/status 2>/dev/null) || continue
    printf "%5d  %4d  %5d  %-20s (RSS: %dMB)\n" \
        "$score" "$adj" "$pid" "$comm" "$((rss/1024))"
done | sort -rn | head -20
EOF
chmod +x /tmp/oom_monitor.sh
$ /tmp/oom_monitor.sh
```

### 실험 3: JVM 컨테이너 메모리 계획 검증

```bash
# 컨테이너에서 JVM 메모리 실제 사용량 측정
$ docker run --memory=2g --name java-test \
  openjdk:21-slim \
  java -XX:+UseContainerSupport \
       -XX:MaxRAMPercentage=75.0 \
       -XX:NativeMemoryTracking=summary \
       -XX:+PrintFlagsFinal \
       -version 2>&1 | grep -E "MaxHeapSize|MaxRAMPercentage"

MaxHeapSize = 1610612736   ← 1.5GB (2GB * 75%)
MaxRAMPercentage = 75.000000

# 실제 JVM 전체 메모리 (Heap + Native) 확인
$ docker stats java-test --no-stream
CONTAINER   MEM USAGE / LIMIT
java-test   1.8GiB / 2GiB  ← Heap 1.5GB + Native 0.3GB = 1.8GB
```

---

## 📊 성능/비용 비교

**oom_score_adj 설정 가이드**:

```
서비스 중요도에 따른 adj 설정 예시:

핵심 인프라 (절대 종료 불가):
  systemd, sshd: adj = -1000 (시스템 기본 설정)

데이터베이스 / 캐시 (중요):
  MySQL, Redis:  adj = -500 ~ -800

웹 서버 (중요):
  Nginx:         adj = -300 ~ -500

애플리케이션 서버:
  Java App:      adj = -100 ~ -300

배치 작업 (희생 가능):
  Batch jobs:    adj = +500 ~ +1000
```

**컨테이너 메모리 limit 설정 권장값**:

```
JVM 서비스 컨테이너 메모리 계획:

  Heap (-Xmx):           컨테이너 limit * 0.70
  Metaspace:             ~256MB ~ 512MB
  Thread Stack:          스레드 수 * 0.5MB
  Code Cache:            ~256MB
  Direct Buffer 등:      ~100MB
  OS/JVM 오버헤드:        ~200MB
  여유:                   ~15%
  -----------------------------------
  컨테이너 limit:         Heap / 0.70
  
  예시: Heap 2GB 필요 → 컨테이너 limit = 2 / 0.70 ≈ 3GB
```

---

## ⚖️ 트레이드오프

**oom_score_adj=-1000 설정의 위험성**

중요 서비스를 OOM 종료에서 완전히 보호하면, 해당 서비스가 메모리를 계속 늘려도 커널이 건드리지 못한다. 다른 모든 프로세스가 죽어 시스템이 불안정해지더라도 해당 서비스는 살아남는다. sshd(원격 접속)를 보호하는 것은 의미있지만, 애플리케이션을 모두 `-1000`으로 설정하면 OOM 상황에서 시스템이 멈출 수 있다.

**컨테이너 OOM 정책 선택**

`memory.oom_kill_disable=1`로 컨테이너 OOM Kill을 비활성화하면 프로세스가 강제 종료되는 대신 메모리 할당이 블로킹된다. 이 경우 컨테이너가 응답 없는 상태로 멈출 수 있다. 일반적으로 OOM Kill을 허용하고 Kubernetes가 Pod를 재시작하도록 두는 것이 낫다.

---

## 📌 핵심 정리

```
OOM Killer 발동 조건:
  물리 메모리 + 스왑 고갈 → 페이지 회수 실패 → OOM

oom_score (0~1000):
  높을수록 먼저 종료
  기반: RSS 크기 / 전체 물리 메모리 * 1000

oom_score_adj (-1000~+1000):
  /proc/<pid>/oom_score_adj 에 쓰기로 조정
  -1000: 절대 종료 안 됨
  +1000: 가장 먼저 종료

JVM vs 커널 OOM 구별:
  종료 코드 137 (= SIGKILL) → 커널 OOM
  종료 코드 1 + 스택 트레이스 → JVM OOM
  확인: dmesg | grep -i oom

컨테이너 JVM 설정:
  -XX:+UseContainerSupport (Java 10+, 필수)
  -XX:MaxRAMPercentage=75.0
  컨테이너 limit = Heap / 0.70 (Heap이 70%가 되도록)

핵심 진단:
  dmesg | grep -i "oom\|killed"
  /proc/<pid>/oom_score        → 현재 점수
  /proc/<pid>/oom_score_adj    → 조정값
  kubectl describe pod → OOMKilled 여부
  exit code 137         → 커널 SIGKILL (OOM)
```

---

## 🤔 생각해볼 문제

**Q1.** Kubernetes Pod에서 `exit code: 137`이 발생했다. OOM Killer가 원인인지 어떻게 확인하는가?

<details>
<summary>해설 보기</summary>

세 가지를 확인한다. 첫째, `kubectl describe pod <name>`에서 `Last State: Reason: OOMKilled`를 확인한다. 둘째, 노드에서 `dmesg -T | grep -i "killed process\|oom"`으로 커널 OOM 로그를 확인한다. 셋째, `kubectl top pod`나 Prometheus `container_memory_working_set_bytes`로 메모리 사용량이 limit에 가까워졌는지 확인한다. exit code 137은 SIGKILL(9)로 종료됐음을 의미하며, OOM 외에도 `kubectl delete pod --grace-period=0`이나 `kill -9`로도 발생할 수 있으므로 반드시 dmesg와 함께 확인해야 한다.

</details>

**Q2.** Redis에 `oom_score_adj=-1000`을 설정했다. 서버 물리 메모리가 완전히 고갈됐을 때 어떤 일이 벌어지는가?

<details>
<summary>해설 보기</summary>

Redis는 OOM Kill에서 보호되지만, 다른 모든 프로세스가 먼저 종료된다. 최종적으로 Redis와 커널만 남게 될 수 있다. 이 시점에 Redis 자체가 새 메모리를 할당하려 하면 할당이 실패한다. `maxmemory` 설정이 있다면 Eviction 정책이 동작하지만, 없다면 Redis가 에러를 반환하거나 새 키 저장에 실패한다. 결국 `-1000` 보호는 Redis를 살리지만 시스템 전체가 불안정해질 수 있다. 중요 서비스 보호와 함께 `maxmemory` 설정으로 Redis 자체 메모리 한도도 반드시 설정해야 한다.

</details>

**Q3.** Java 애플리케이션에서 `java.lang.OutOfMemoryError: Direct buffer memory` 오류가 발생했다. 이것은 JVM Heap 문제인가, 커널 OOM인가?

<details>
<summary>해설 보기</summary>

JVM 내부 오류이지만, Heap이 아닌 Direct Memory(Native Memory) 문제다. `ByteBuffer.allocateDirect()`로 할당하는 Direct Buffer는 JVM Heap 외부의 Native 메모리를 사용한다. JVM이 `-XX:MaxDirectMemorySize` 한도(기본: Heap 크기와 동일)를 초과하면 이 오류가 발생한다. 커널 OOM이 아니므로 dmesg에는 아무것도 없다. 해결 방법은 `-XX:MaxDirectMemorySize` 값을 늘리거나, Direct Buffer를 명시적으로 해제하는 것이다. Netty 같은 프레임워크는 Direct Buffer를 대량으로 사용하므로 이 설정이 중요하다.

</details>

---

**[⬅️ 이전: 메모리 할당기](./05-memory-allocator.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — 파일 디스크립터와 시스템 콜 ➡️](../io-models/01-file-descriptor-syscall.md)**
