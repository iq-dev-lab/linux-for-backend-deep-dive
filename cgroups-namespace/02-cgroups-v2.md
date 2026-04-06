# 02. cgroups v2 — 자원 제한의 실체

## 🎯 핵심 질문

- `docker run --memory=512m --cpus=1.5`가 커널에 실제로 쓰는 파일은 무엇인가?
- CPU CFS 쿼터로 CPU를 제한할 때 "Throttling"이 발생하는 메커니즘은?
- 메모리 한도 초과 시 커널이 페이지 회수와 OOM Kill 중 무엇을 먼저 하는가?
- cgroups v1과 v2의 핵심 차이는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**`top`에서 CPU 사용률 40%인데 응답이 느리다.** cgroups CPU Throttling 때문이다. `--cpus=0.5`로 제한된 컨테이너가 쿼터를 조기 소진하면 남은 시간 동안 강제 대기한다. CPU 사용률이 낮아 보여도 응답 시간이 늘어나는 이유가 여기에 있다.

**Kubernetes `resources.limits.memory`를 초과하면 Pod가 왜 재시작되는가?** 커널 OOM Killer가 cgroups 메모리 한도를 초과한 프로세스를 종료한다. PID 1이 종료되면 컨테이너가 종료되고 Kubernetes가 Pod를 재시작한다. `kubectl describe pod`에서 `OOMKilled` 이유가 표시된다.

**컨테이너 100개를 실행했더니 호스트 메모리가 부족하다.** cgroups 메모리 제한을 설정하지 않으면 컨테이너가 호스트 메모리를 무제한 사용할 수 있다. `requests`와 `limits`를 모두 설정하는 것이 Kubernetes에서 권장되는 이유다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: CPU Throttling을 CPU 사용률로 진단
$ docker stats mycontainer
CONTAINER   CPU %   MEM USAGE
mycontainer  45%    400MB / 512MB

# "CPU 45%면 여유 있겠지" → 잘못된 판단
# --cpus=0.5로 제한됐다면 50ms/100ms 쿼터
# 45ms 쓰고 남은 55ms 대기 → CPU 45%인데 응답 느림

# cpu.stat으로 확인:
$ cat /sys/fs/cgroup/cpu/docker/<id>/cpu.stat
nr_periods 1000
nr_throttled 450   ← 45% 스로틀링! (CPU % 아님)
throttled_time 22500000000  ← 나노초 단위

# 실수 2: memory limit과 실제 사용량 혼동
$ docker run --memory=512m app
# OOM Kill 발생 후 "왜? 512MB 줬는데"
# → JVM Heap 512MB + Metaspace 256MB + 기타 = 실제 800MB 필요
# → limit 512MB < 실제 필요 800MB → OOM

# 실수 3: cgroups v1과 v2 혼용
$ ls /sys/fs/cgroup/
cgroup.controllers  cgroup.procs  memory.max  ...  ← v2 (단일 계층)
# 또는
$ ls /sys/fs/cgroup/
blkio  cpu  cpuacct  memory  ...  ← v1 (분리된 계층)
# 두 버전에서 파일 이름과 경로가 다름
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# Docker 컨테이너의 cgroups 경로 찾기
$ CONTAINER_ID=$(docker inspect --format '{{.Id}}' mycontainer)

# cgroups v2 경로
$ CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope"

# 메모리 제한 확인
$ cat $CGROUP_PATH/memory.max
536870912   ← 512MB (512 * 1024 * 1024)

# CPU 제한 확인
$ cat $CGROUP_PATH/cpu.max
150000 100000  ← 쿼터(150ms) / 주기(100ms) = 1.5 CPU

# 실시간 CPU Throttling 모니터링
$ watch -n 1 "cat $CGROUP_PATH/cpu.stat"
usage_usec 12345678
user_usec 9876543
system_usec 2469135
nr_periods 1000
nr_throttled 150    ← 15% 스로틀링
throttled_usec 750000

# 메모리 상세 현황
$ cat $CGROUP_PATH/memory.current   # 현재 사용량
$ cat $CGROUP_PATH/memory.max       # 하드 한도
$ cat $CGROUP_PATH/memory.high      # 소프트 한도 (throttle 시작)
$ cat $CGROUP_PATH/memory.stat      # 상세 분류
```

---

## 🔬 내부 동작 원리

### cgroups의 두 버전 비교

```
cgroups v1 (레거시):
  /sys/fs/cgroup/
  ├── cpu/         → CPU 제어
  │   └── docker/
  │       └── <id>/
  │           ├── cpu.cfs_quota_us
  │           └── cpu.cfs_period_us
  ├── memory/      → 메모리 제어
  │   └── docker/
  │       └── <id>/
  │           └── memory.limit_in_bytes
  └── blkio/       → 블록 I/O 제어

  문제: 컨트롤러가 분리되어 계층 구조가 다름
       프로세스가 여러 cgroup에 동시에 속할 수 있어 복잡

cgroups v2 (현대):
  /sys/fs/cgroup/
  └── system.slice/
      └── docker-<id>.scope/
          ├── memory.max        ← 메모리 하드 한도
          ├── memory.high       ← 메모리 소프트 한도
          ├── cpu.max           ← CPU 쿼터/주기
          ├── cpu.weight        ← CPU 상대적 가중치
          ├── io.max            ← I/O 대역폭 한도
          └── cgroup.procs      ← 소속 프로세스 목록

  장점: 단일 계층 구조, 일관된 인터페이스
       프로세스는 정확히 하나의 cgroup에 속함
```

### docker run --memory=512m의 커널 동작

```
$ docker run --memory=512m --cpus=1.5 myapp

Docker daemon → containerd → runc:

1. namespace 생성 (PID, Network, Mount, UTS, IPC)
2. cgroup 생성:
   mkdir /sys/fs/cgroup/system.slice/docker-<id>.scope

3. 메모리 제한 설정:
   echo 536870912 > memory.max   (512 * 1024 * 1024 = 536870912)
   echo 536870912 > memory.high  (소프트 한도, 보통 max와 동일)

4. CPU 제한 설정:
   echo "150000 100000" > cpu.max
   # 150000 = 쿼터 (150ms per 100ms 주기 = 1.5 CPU)

5. 프로세스를 cgroup에 추가:
   echo <pid> > cgroup.procs

6. 컨테이너 프로세스 실행

검증:
$ cat /sys/fs/cgroup/system.slice/docker-<id>.scope/memory.max
536870912

$ cat /sys/fs/cgroup/system.slice/docker-<id>.scope/cpu.max
150000 100000
```

### CPU CFS Throttling 상세 메커니즘

```
CPU CFS (Completely Fair Scheduler) Bandwidth Control:

설정: cpu.max = "150000 100000"
  쿼터 = 150000 μs (150ms)
  주기 = 100000 μs (100ms)
  허용 CPU = 150ms / 100ms = 1.5 CPU

100ms 주기 동안의 동작:

  시간:  0ms     25ms    50ms    75ms    100ms(다음 주기)
         ┌───────┐       ┌───────┐       ┌───────
  실행:   │ 75ms  │       │ 75ms  │       │ ...
         └───────┘       └───────┘       └───────
  스로틀: 0ms   [스로틀 25ms] [25ms]    0ms

  4코어 병렬 실행:
    100ms * 4코어 = 400ms 물리 시간
    쿼터 150ms 소진 시점: 37.5ms (150ms / 4코어)
    스로틀링 기간: 100ms - 37.5ms = 62.5ms

  이것이 "CPU 40%인데 응답 느린" 이유:
    실제로 37.5ms 실행, 62.5ms 강제 대기
    외부에서 보면 CPU 사용률 = 37.5/100 = 37.5%
    응답 시간 = 최악의 경우 62.5ms 추가 지연

Throttling 해결:
  1. --cpus 값 늘리기 (ex: 0.5 → 1.0)
  2. 애플리케이션 스레드 수 줄이기 (멀티스레드 → 병렬 실행 감소)
  3. cfs_period_us 줄이기 (주기 짧게 → 스로틀링 짧아짐)
     echo "50000 50000" > cpu.max  (50ms/50ms = 1 CPU)
```

### 메모리 한도 초과 시 커널 동작 순서

```
메모리 요청 → 한도 초과 시:

1단계: 페이지 회수 (Reclaim)
  memory.high 초과:
    → 해당 cgroup의 프로세스를 느리게 만들며 페이지 회수
    → 프로세스는 계속 실행 (죽지 않음)
    → Page Cache, 익명 페이지 LRU로 회수

  memory.max 초과:
    → 즉시 강제 페이지 회수 시도
    → 회수 성공: 메모리 확보 후 계속 실행

2단계: OOM Kill (회수 실패 시)
  해당 cgroup 내에서 oom_score가 높은 프로세스 선택
  → SIGKILL 전송 → 프로세스 종료
  → 보통 메모리 가장 많이 쓰는 프로세스

  Kubernetes OOMKilled:
    PID 1 종료 → 컨테이너 종료 → Pod 재시작

  dmesg 확인:
    Memory cgroup out of memory: Kill process 1234 (java) score 1000

memory.oom_control (v1) / memory.oom.kill_disable (v2):
  OOM Kill 비활성화 가능 (프로세스가 멈추는 대신)
  → 주의: 응답 없는 컨테이너 상태 가능
```

---

## 💻 실전 실험

### 실험 1: cgroups 값 직접 확인

```bash
# 제한 있는 컨테이너 실행
$ docker run -d --name cg-test \
  --memory=256m --cpus=0.5 \
  --memory-swap=256m \  # 스왑도 제한
  ubuntu sleep infinity

$ CGPATH=$(find /sys/fs/cgroup -name "*.scope" | \
  xargs grep -l "$(docker inspect --format '{{.State.Pid}}' cg-test)" \
  2>/dev/null | head -1 | xargs dirname)

# 또는 cgroups v2에서:
$ CGPATH=$(cat /proc/$(docker inspect --format '{{.State.Pid}}' cg-test)/cgroup \
  | head -1 | cut -d: -f3)
$ CGPATH="/sys/fs/cgroup$CGPATH"

echo "메모리 하드 한도: $(cat $CGPATH/memory.max)"
echo "메모리 현재 사용: $(cat $CGPATH/memory.current)"
echo "CPU 제한: $(cat $CGPATH/cpu.max)"
echo "CPU 사용 통계:"
cat $CGPATH/cpu.stat
```

### 실험 2: CPU Throttling 재현 및 측정

```bash
# CPU 0.5개로 제한 + 멀티스레드 부하
$ docker run -d --name throttle-test --cpus=0.5 ubuntu \
  bash -c 'apt-get install -y stress-ng -q && stress-ng --cpu 2 --timeout 30s'

# Throttling 실시간 모니터링
$ CPID=$(docker inspect --format '{{.State.Pid}}' throttle-test)
$ CGPATH=$(cat /proc/$CPID/cgroup | grep "^0:" | cut -d: -f3)
$ watch -n 0.5 "cat /sys/fs/cgroup${CGPATH}/cpu.stat | grep -E 'nr_|throttled'"

# 예상 출력:
# nr_periods 100
# nr_throttled 65    ← 65% 스로틀링 (2코어가 0.5 쿼터 경쟁)
# throttled_usec 65000000

# docker stats와 비교
$ docker stats throttle-test --no-stream
CONTAINER    CPU %   MEM USAGE
throttle-test  49%   50MB / unlimited
# CPU 49%이지만 실제로는 65% 스로틀링 중!
```

### 실험 3: 메모리 한도 초과 OOM 재현

```bash
# 작은 메모리 한도로 컨테이너 실행
$ docker run --name oom-test --memory=64m ubuntu \
  bash -c 'dd if=/dev/zero bs=1M | head -c 100M > /dev/null'

# OOM Kill 발생 후 종료 코드 확인
$ docker inspect oom-test --format '{{.State.ExitCode}}'
137  ← SIGKILL(9) = OOM Kill

$ docker inspect oom-test --format '{{.State.OOMKilled}}'
true  ← OOM Kill 확인!

# 호스트 dmesg에서 OOM 로그
$ dmesg | grep -i "oom\|killed" | tail -5
[...] Memory cgroup out of memory: Kill process 1234 (dd) score 1000
[...] Killed process 1234 (dd) total-vm:102400kB, anon-rss:65536kB
```

---

## 📊 성능/비용 비교

**CPU Throttling 발생 조건**:

```
--cpus 설정   스레드 수   스로틀링 발생?
0.5           1           아니오 (50ms/100ms, 여유 있음)
0.5           2           예! (2스레드 * 50ms = 100ms > 쿼터 50ms)
1.0           4           예! (4스레드 * 25ms = 100ms > 쿼터 100ms)
2.0           2           아니오 (2스레드 * 100ms = 200ms ≤ 쿼터 200ms)

실무 원칙:
  --cpus 값 ≥ 실제로 병렬 실행되는 스레드 수 * 사용률
  안전 여유: 20~30% 추가
```

**cgroups v1 vs v2 파일 경로 차이**:

```
리소스    cgroups v1 경로                         cgroups v2 경로
메모리    /sys/fs/cgroup/memory/.../              /sys/fs/cgroup/.../
한도      memory.limit_in_bytes                   memory.max
현재 사용  memory.usage_in_bytes                  memory.current
통계      memory.stat                             memory.stat

CPU 한도  cpu.cfs_quota_us + cpu.cfs_period_us    cpu.max (단일 파일)
CPU 가중치 cpu.shares                              cpu.weight
스로틀    cpu.stat (nr_throttled)                 cpu.stat (nr_throttled)
```

---

## ⚖️ 트레이드오프

**memory.high vs memory.max**

`memory.high`는 소프트 한도로, 초과 시 프로세스를 스로틀링하며 메모리를 회수한다. 프로세스는 살아있지만 느려진다. `memory.max`는 하드 한도로, 초과 시 OOM Kill이 발생한다. 운영 환경에서 `memory.high`를 `memory.max`의 80% 수준으로 설정하면 OOM Kill 전에 경고 메커니즘으로 활용할 수 있다.

**CPU 쿼터 주기(cfs_period_us) 조정**

주기를 짧게 하면(예: 10ms → 1ms) 스로틀링이 더 촘촘하게 발생해 레이턴시 변동이 줄어든다. 하지만 스케줄러가 더 자주 개입해 오버헤드가 증가한다. 레이턴시에 민감한 서비스는 짧은 주기가, 처리량 중심 서비스는 긴 주기가 유리하다.

---

## 📌 핵심 정리

```
docker run --memory=512m --cpus=1.5 → 커널에서:
  memory.max = 536870912 (bytes)
  cpu.max = "150000 100000"  (150ms/100ms = 1.5 CPU)

CPU Throttling 원리:
  CFS 쿼터 소진 → 남은 주기 동안 강제 대기
  CPU% 낮은데 응답 느리면 → cpu.stat nr_throttled 확인

메모리 한도 초과 처리 순서:
  1. memory.high 초과 → 프로세스 스로틀 + 페이지 회수
  2. memory.max 초과 → 강제 회수
  3. 회수 실패 → OOM Kill → exit code 137

cgroups v2 핵심 파일 (단일 계층):
  memory.max     → 메모리 하드 한도
  memory.current → 현재 사용량
  cpu.max        → CPU 쿼터/주기
  cpu.stat       → CPU 통계 (스로틀링 포함)
  cgroup.procs   → 소속 프로세스

핵심 진단:
  cpu.stat nr_throttled          → CPU 스로틀링 발생 횟수
  docker inspect OOMKilled true  → OOM Kill 여부
  exit code 137 (=128+9)         → 커널 SIGKILL (OOM)
  dmesg | grep oom               → 커널 OOM 로그
```

---

## 🤔 생각해볼 문제

**Q1.** Kubernetes `resources.requests.cpu=0.5`와 `resources.limits.cpu=1.0`의 차이는 cgroups 레벨에서 어떻게 구현되는가?

<details>
<summary>해설 보기</summary>

`requests.cpu=0.5`는 cgroups의 `cpu.weight`(또는 v1의 `cpu.shares`)로 구현된다. 이것은 CPU 경합 시 상대적 가중치이며, CPU 여유가 있으면 limits까지 더 쓸 수 있다. `limits.cpu=1.0`은 `cpu.max`의 쿼터로 구현된다(`100000 100000` = 1 CPU). 요청이 1CPU를 초과하면 스로틀링이 발생한다. 즉, requests는 최소 보장(가중치 기반)이고, limits는 절대 상한(CFS 쿼터)이다.

</details>

**Q2.** 컨테이너 메모리 사용량을 `memory.current`로 확인했는데 `memory.max`보다 훨씬 작다. 그런데 OOM이 발생했다. 어떻게 가능한가?

<details>
<summary>해설 보기</summary>

두 가지 가능성이 있다. 첫째, `memory.current`는 Page Cache를 포함한 전체 사용량인데, OOM 시점에 익명 메모리(Heap 등)가 급격히 늘어났을 수 있다. `memory.stat`에서 `anon` 값을 확인해야 한다. 둘째, JVM이 `mmap()`으로 Native Memory를 요청했는데 이것이 `memory.current`에 즉시 반영되지 않는 경우다(Demand Paging). 실제 물리 페이지 접근 시점에 한도 초과가 발생한다. `memory.stat`의 `pgfault`와 `pgmajfault`를 모니터링하면 실제 메모리 접근 패턴을 확인할 수 있다.

</details>

**Q3.** `--memory-swap=512m`과 `--memory=256m`을 함께 설정하면 스왑은 얼마나 사용 가능한가?

<details>
<summary>해설 보기</summary>

스왑 사용 가능량은 `memory-swap - memory = 512m - 256m = 256m`이다. `--memory-swap`은 메모리 + 스왑의 합계 한도를 의미한다. `--memory-swap=-1`로 설정하면 스왑을 무제한 사용할 수 있다. `--memory-swap`을 `--memory`와 같게 설정하면 스왑을 전혀 사용하지 않는다(`--memory-swap=256m --memory=256m`이면 스왑 0). 스왑 없이 운영하면 메모리 초과 시 즉시 OOM Kill이 발생하고, 스왑을 허용하면 성능 저하(스왑 I/O)를 감수하고 더 버틸 수 있다.

</details>

---

**[⬅️ 이전: Linux namespace](./01-linux-namespace.md)** | **[홈으로 🏠](../README.md)** | **[다음: 컨테이너 네트워킹 ➡️](./03-container-networking.md)**
