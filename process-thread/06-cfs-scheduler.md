# 06. CFS 스케줄러 — 공정한 CPU 할당의 원리

## 🎯 핵심 질문

- CFS(Completely Fair Scheduler)는 어떤 기준으로 다음 실행할 프로세스를 선택하는가?
- `nice` 값이 CPU 사용 시간에 미치는 영향은 수치로 얼마인가?
- cgroups의 CPU 쿼터 제한이 CFS 위에서 어떻게 동작하는가?
- CPU 어피니티(affinity)를 설정하면 성능이 왜 향상될 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Docker `--cpus=1.5` 설정 시 컨테이너가 간헐적으로 응답이 느려진다.** cgroups CPU 쿼터가 CFS 스케줄러 위에서 동작하기 때문이다. 쿼터를 초과하면 컨테이너의 프로세스가 **CPU Throttling** 상태가 되어 타임슬라이스를 강제로 빼앗긴다. 이를 이해하지 못하면 CPU 사용률이 낮은데도 응답이 느린 이유를 찾을 수 없다.

**백그라운드 배치 작업이 웹 서버 응답에 영향을 준다.** 같은 서버에서 배치 작업이 CPU를 많이 쓰면 웹 서버 응답이 느려진다. `nice` 값으로 배치 작업의 우선순위를 낮추면 CFS가 웹 서버에 더 많은 CPU 시간을 할당한다.

**Redis, Kafka 같은 지연 민감 서비스를 특정 CPU 코어에 고정해야 한다.** CPU 어피니티로 캐시 지역성을 높이고 NUMA 경계를 피해 지연을 일관되게 유지할 수 있다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: CPU 사용률이 낮은데 응답이 느린 현상
$ docker stats my-container
CONTAINER   CPU %   MEM USAGE
my-app      30%     500MB / 2GB

# "CPU 30%인데 왜 느리지?"
# → CPU Throttling 확인 안 함
# → 실제로 쿼터 100ms 중 30ms 사용 + 70ms 대기 중
$ cat /sys/fs/cgroup/cpu/docker/<id>/cpu.stat
nr_throttled: 1234      ← 스로틀링 발생 횟수
throttled_time: 5000000 ← 누적 스로틀링 시간(나노초)
# → 명백한 CPU 스로틀링이지만 CPU% 지표만 봐서 놓침

# 실수 2: 배치 작업 우선순위 무관리
$ java -jar batch-job.jar &
# → nice 0 (기본)으로 실행
# → 웹 서버와 동등한 CPU 경쟁
# → 배치 실행 중 웹 응답 지연
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# CPU Throttling 모니터링
$ watch -n 1 'cat /sys/fs/cgroup/cpu/docker/$(docker ps -q --filter name=my-app)/cpu.stat'

# Prometheus + cAdvisor로 쿠버네티스 CPU Throttling 확인
# container_cpu_cfs_throttled_periods_total 메트릭

# 배치 작업 우선순위 낮추기
$ nice -n 19 java -jar batch-job.jar
# 또는 실행 중인 프로세스의 nice 값 변경
$ renice -n 19 -p $BATCH_PID

# CPU 어피니티 설정 (CPU 0-3에 고정)
$ taskset -c 0-3 redis-server
# 또는 cgroups로 설정
$ docker run --cpuset-cpus="0,1" redis
```

---

## 🔬 내부 동작 원리

### CFS의 핵심 개념: vruntime

CFS는 모든 태스크가 CPU 시간을 "공평하게" 사용하는 것을 목표로 한다. 핵심은 `vruntime`(virtual runtime, 가상 실행 시간)이다.

```
vruntime 계산:
  vruntime += 실제 실행 시간 * (기준 가중치 / 태스크 가중치)

  기준 가중치(nice=0): 1024
  nice = -20 (높은 우선순위): 가중치 = 88761
  nice =   0 (기본):         가중치 = 1024
  nice = +19 (낮은 우선순위): 가중치 = 15

예시:
  태스크 A (nice=0, 가중치 1024): 1ms 실행
    vruntime += 1ms * (1024/1024) = 1ms

  태스크 B (nice=-5, 가중치 3121): 1ms 실행
    vruntime += 1ms * (1024/3121) ≈ 0.33ms  ← vruntime 증가 느림!

  CFS는 항상 vruntime이 가장 작은 태스크를 선택
  → B의 vruntime이 더 천천히 증가
  → B가 더 자주 선택됨 = 더 많은 CPU 시간 획득
```

### Red-Black Tree 자료구조

CFS는 실행 가능한 태스크를 `vruntime`을 키로 하는 **레드-블랙 트리**로 관리한다.

```
레드-블랙 트리 (vruntime 기준 정렬):

             [vruntime=50]
            /              \
      [vruntime=30]    [vruntime=70]
      /          \
[vruntime=10] [vruntime=40]
      ↑
  leftmost: 항상 이 태스크를 선택 (O(1) 조회)

태스크 선택: O(1) - 트리의 가장 왼쪽 노드
태스크 삽입: O(log N) - 트리 삽입
타임슬라이스 소진 후: vruntime 증가 → 트리 내 위치 변경
```

### nice 값과 CPU 사용 비율

```
nice 값에 따른 CPU 가중치 (리눅스 커널 sched_prio_to_weight 테이블):

nice | 가중치  | nice=0 대비 비율
-20  | 88761  | 86.7x
-15  | 29154  | 28.5x
-10  |  9548  |  9.3x
 -5  |  3121  |  3.0x
  0  |  1024  |  1.0x  (기본)
  5  |   335  |  0.33x
 10  |   110  |  0.11x
 15  |    36  |  0.035x
 19  |    15  |  0.015x

예시: nice=0 태스크와 nice=5 태스크 동시 실행
  CPU 배분: 1024 / (1024+335) ≈ 76% : 24%
```

### cgroups CPU 제한 (CFS Bandwidth Control)

```
Docker --cpus=1.5 설정 시:

/sys/fs/cgroup/cpu/docker/<id>/
  cpu.cfs_period_us: 100000  ← 100ms 주기
  cpu.cfs_quota_us:  150000  ← 150ms 사용 허용 (= 1.5 CPU)

동작 방식:
  주기 0~100ms: 컨테이너가 150ms분 CPU 사용 가능
  → 4코어 병렬로 사용 시 37.5ms만에 쿼터 소진
  → 나머지 62.5ms: CPU Throttling (강제 대기!)

  주기 100~200ms: 쿼터 리셋 → 다시 실행

CPU 사용률 30%처럼 보이지만 실제 동작:
  [실행 37.5ms] [스로틀링 62.5ms] [실행 37.5ms] [스로틀링 62.5ms] ...
                 ↑↑↑↑↑↑↑↑↑↑↑↑
               이 구간에 요청이 오면 응답 지연!
```

```bash
# CPU Throttling 지표 해석
$ cat /sys/fs/cgroup/cpu/docker/<id>/cpu.stat
nr_periods: 1000           ← 총 CFS 주기 수
nr_throttled: 300          ← 스로틀링된 주기 수
throttled_time: 18000000000 ← 누적 스로틀링 시간(나노초)

# 스로틀링 비율 = nr_throttled / nr_periods = 300/1000 = 30%
# → 전체 시간의 30%를 스로틀링으로 낭비
```

### CPU 어피니티와 NUMA

```
NUMA(Non-Uniform Memory Access) 아키텍처:

  [소켓 0]              [소켓 1]
  코어 0-7              코어 8-15
  메모리 뱅크 A          메모리 뱅크 B
      │                      │
      └──────────────────────┘
              QPI 링크 (느림)

프로세스가 코어 0에서 실행 중 → 소켓 1 메모리 접근:
  코어 0 → QPI → 소켓 1 메모리  ← NUMA 원격 접근 (느림!)

CPU 어피니티 설정:
  taskset -c 0-7 redis-server
  → 소켓 0 코어에만 고정
  → 항상 소켓 0 메모리(로컬) 접근
  → NUMA 원격 접근 제거

캐시 지역성 효과:
  특정 코어에 고정 → L1/L2 캐시에 데이터 유지
  코어가 계속 바뀌면 → 매번 캐시 콜드 스타트
```

---

## 💻 실전 실험

### 실험 1: nice 값 효과 측정

```bash
# CPU 집약적 태스크 두 개 동시 실행 후 CPU 배분 비교
$ stress-ng --cpu 1 --cpu-method fft &
$ PID1=$!

# nice=0으로 동일 태스크 실행
$ nice -n 0 stress-ng --cpu 1 --cpu-method fft &
$ PID2=$!

$ pidstat -u 1 5 -p $PID1,$PID2
# 예상: 거의 50:50 배분

# 한쪽을 nice=10으로 변경
$ renice -n 10 -p $PID1

$ pidstat -u 1 5 -p $PID1,$PID2
# 예상: nice=0이 ~90% / nice=10이 ~10% 배분
# (1024 / (1024+110) ≈ 90%)

kill $PID1 $PID2
```

### 실험 2: CPU Throttling 재현

```bash
# CPU 쿼터 0.5 CPU로 제한된 컨테이너
$ docker run -d --name throttle-test --cpus=0.5 \
  ubuntu:22.04 sleep infinity

$ docker exec -d throttle-test stress-ng --cpu 2 --timeout 30s

# 스로틀링 상태 확인
$ CONTAINER_ID=$(docker inspect --format='{{.Id}}' throttle-test)
$ watch -n 0.5 "cat /sys/fs/cgroup/cpu/docker/${CONTAINER_ID}/cpu.stat"

# nr_throttled가 증가하는 것을 관찰
# throttled_time도 빠르게 증가

# Docker stats로 CPU 사용률과 비교
$ docker stats throttle-test
# CPU% ≈ 50% (0.5 CPU 제한)
# 실제로는 0.5/1 주기 동안 실행, 나머지 대기
```

### 실험 3: 스케줄링 지연 측정

```bash
# cyclictest로 스케줄링 지연 측정
$ apt-get install -y rt-tests
$ cyclictest -l 100000 -m -n -p 80 -i 200 -h 400

# CFS 기본 스케줄러:
# T: 0 (52):  0 Avg: 15 Max: 52  ← 평균 15μs, 최대 52μs 지연

# nice 값과 스케줄링 지연 관계 확인
$ nice -n -20 cyclictest -l 100000 -m -n -i 200 -h 400
# 더 낮은 지연 예상
```

---

## 📊 성능/비용 비교

**nice 값에 따른 CPU 시간 배분 (동일 태스크 2개 동시 실행)**:

```
태스크 A nice | 태스크 B nice | A:B CPU 배분
           0  |            0  | 50% : 50%
           0  |            5  | 75% : 25%
           0  |           10  | 90% : 10%
           0  |           19  | 98% :  2%
          -5  |            0  | 75% : 25%
         -10  |            0  | 90% : 10%
```

**cgroups CPU 제한 vs 물리 CPU 수에 따른 스로틀링**:

```
설정: --cpus=1.0, 물리 4코어

싱글 스레드 앱 (CPU 사용률 ≤ 100%):
  → 스로틀링 없음 (1 CPU = 1코어 100%)

멀티 스레드 앱 (4스레드 병렬):
  → 25ms마다 쿼터 소진 (4코어 * 25ms = 100ms)
  → 75ms 스로틀링
  → 스로틀링 비율: 75%

해결: --cpus를 실제 병렬 스레드 수에 맞게 설정
```

---

## ⚖️ 트레이드오프

**CFS 공정성 vs 응답 지연**:

CFS는 모든 태스크에 공정한 CPU 시간을 할당하려 한다. 하지만 지연 민감 서비스(Redis, 실시간 데이터 처리 등)에서는 공정성보다 낮은 지연이 중요할 수 있다. 이 경우 `SCHED_FIFO`나 `SCHED_RR` 실시간 스케줄링 정책 또는 높은 우선순위(낮은 nice 값)를 사용할 수 있다.

**CPU 어피니티의 장단점**:

CPU 어피니티는 캐시 지역성을 높이고 NUMA 원격 접근을 피하게 해 지연을 일관되게 유지한다. 단점은 특정 코어에만 부하가 집중되고, 시스템 전체 CPU 활용률이 낮아질 수 있다. Redis처럼 단일 스레드 이벤트 루프를 가진 서비스에는 유용하지만, 멀티 스레드 서비스에는 오히려 역효과가 날 수 있다.

**CPU Throttling 해결 접근법**:

`--cpus` 값을 늘리는 것이 직관적이지만, 비용 측면을 고려해야 한다. 대안으로 `cpu.cfs_period_us`를 줄이면(예: 100ms → 10ms) 같은 쿼터를 더 자주 분배해 스로틀링 기간이 짧아진다. 지연은 줄지만 스케줄러 오버헤드가 증가한다.

---

## 📌 핵심 정리

```
CFS 핵심:
  vruntime이 가장 작은 태스크를 선택
  가중치(nice 기반)로 vruntime 증가 속도 조절
  → nice 낮을수록 가중치 높음 → vruntime 천천히 증가 → 더 자주 선택

nice 값 범위: -20(최고 우선순위) ~ +19(최저 우선순위)
  nice 조정: nice -n 19 command  또는  renice -n 19 -p PID

cgroups CPU 제한:
  cpu.cfs_quota_us / cpu.cfs_period_us = 허용 CPU 수
  쿼터 소진 → CPU Throttling → 응답 지연
  진단: /sys/fs/cgroup/cpu/.../cpu.stat의 nr_throttled

CPU 어피니티:
  taskset -c 0-3 command   → CPU 코어 0-3에 고정
  → 캐시 지역성 향상, NUMA 원격 접근 방지

핵심 진단 명령어:
  pidstat -u 1     → 프로세스별 CPU 사용률
  ps -eo pid,ni,comm → nice 값 확인
  renice -n X -p PID → 실행 중 nice 변경
  cpu.stat         → cgroups CPU Throttling 여부
```

---

## 🤔 생각해볼 문제

**Q1.** Kubernetes에서 Pod의 CPU Request와 Limit을 어떻게 설정해야 CPU Throttling을 최소화할 수 있는가?

<details>
<summary>해설 보기</summary>

**Request와 Limit을 동일하게 설정**(Guaranteed QoS 클래스)하거나, Limit을 Request의 2~3배 이내로 설정하는 것이 권장된다. 중요한 것은 실제 피크 CPU 사용량을 측정하고 Limit을 그에 맞게 설정하는 것이다. Limit을 지나치게 낮게 설정하면 CPU 사용률이 낮아 보여도 내부적으로 스로틀링이 발생한다. `container_cpu_cfs_throttled_periods_total` 메트릭으로 스로틀링 비율을 모니터링하고, 스로틀링 비율이 25%를 넘으면 Limit 상향을 검토한다.

</details>

**Q2.** Redis를 4코어 서버에서 실행할 때 CPU 어피니티로 단일 코어에 고정하는 것이 좋은가, 4코어 모두 허용하는 것이 좋은가?

<details>
<summary>해설 보기</summary>

Redis의 주 이벤트 루프는 단일 스레드다. 단일 코어에 고정하면 L1/L2 캐시 히트율이 높아지고 캐시 미스가 줄어 지연이 낮아진다. 반면 4코어 허용 시 스케줄러가 Redis를 여러 코어로 옮길 수 있어 캐시 워밍업 비용이 반복된다. Redis 공식 문서도 CPU 어피니티 설정을 권장한다. 단, I/O 스레드(Redis 6.0+)나 백그라운드 스레드는 별도 코어에 고정하는 것이 이상적이다. `taskset -c 0 redis-server`로 메인 스레드를 코어 0에, I/O 스레드를 코어 1-2에 설정하는 방식을 고려할 수 있다.

</details>

**Q3.** `top`에서 Java 프로세스의 CPU 사용률이 0%인데도 요청이 느리다. 어떻게 진단하는가?

<details>
<summary>해설 보기</summary>

CPU Throttling을 의심해야 한다. `/sys/fs/cgroup/cpu/docker/<id>/cpu.stat`에서 `nr_throttled`와 `throttled_time`을 확인한다. Throttling이 확인되면 `--cpus` 제한을 늘린다. CPU 0%인데 느린 다른 원인으로는 I/O 대기(`wa` 수치 확인), 메모리 스왑(`vmstat`의 `si/so`), GC 일시 정지(`jstat -gcutil`로 GC 시간 확인)도 있다. `jstack`으로 스레드 덤프를 확인해 모든 스레드가 `BLOCKED` 또는 `WAITING` 상태인지도 확인한다.

</details>

---

**[⬅️ 이전: 시그널과 IPC](./05-signal-ipc.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — 가상 메모리 ➡️](../memory-management/01-virtual-memory.md)**
