# 01. CPU 분석 — us/sy/wa/id의 의미

## 🎯 핵심 질문

- `top`의 `us`, `sy`, `wa`, `id` 각 항목이 커널 레벨에서 무엇을 의미하는가?
- `sy`가 높을 때와 `wa`가 높을 때 다음 확인 단계는 무엇인가?
- Load Average가 CPU 코어 수를 초과하면 정확히 어떤 일이 벌어지는가?
- `mpstat -P ALL`로 특정 코어에 부하가 몰리는 현상을 어떻게 진단하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**CPU 사용률 70%인데 응답이 느리다.** `us`인지 `sy`인지 `wa`인지에 따라 원인과 해결 방법이 완전히 다르다. `us`(유저) 70%라면 애플리케이션 로직, `sy`(시스템) 70%라면 과도한 시스템 콜, `wa`(I/O 대기) 70%라면 디스크나 네트워크 병목이다.

**Load Average가 "4.5"인데 CPU가 4코어다.** 평균 4.5개의 태스크가 CPU를 기다리고 있다는 뜻이다. 50%의 태스크가 대기 중이므로 응답이 느려질 수 있다. `wa`가 높으면 I/O를 기다리는 것이고, `id`가 낮으면 순수 CPU 경쟁이다.

**Redis 서버에서 `sy`가 갑자기 20%로 올랐다.** 단일 스레드 이벤트 루프인 Redis에서 `sy`가 높다는 것은 시스템 콜이 많아졌다는 뜻이다. `strace -c`로 어떤 시스템 콜이 폭발했는지 확인해야 한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: CPU % 하나만 보고 판단
$ top
%Cpu(s): 15.0 us, 25.0 sy, 0.0 ni, 55.0 id, 5.0 wa, 0.0 hi, 0.0 si

# "CPU 40% 사용 중이네" → 잘못된 해석
# sy 25%는 매우 높은 수치! 시스템 콜 과다 의심
# wa 5%는 I/O 대기 있음을 시사

# 실수 2: Load Average의 의미 오해
$ uptime
load average: 8.5, 7.2, 6.1
# "CPU 8코어인데 Load Average 8.5면 괜찮겠지?"
# → 8.5 = 평균 8.5개 태스크가 대기 or 실행 중
# → 8코어에서 8.5 = 일부 태스크가 항상 대기
# → wa vs sy 중 무엇 때문인지 구분 필요

# 실수 3: 특정 코어 편중 미확인
$ top
%Cpu(s): 12.5 us, 5.0 sy, 0.0 wa
# "CPU 17.5% 전체, 괜찮네" → 평균이라 놓침
# 실제로는 코어 0 하나가 100%, 나머지 7개는 0%
$ mpstat -P ALL 1  # 코어별 확인 필요
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# CPU 상태 단계별 진단

# Step 1: 전체 현황 파악
$ top
%Cpu(s): 5.0 us, 30.0 sy, 0.0 ni, 60.0 id, 5.0 wa
# sy 30%! → 다음 단계로

# Step 2: sy 높음 → 시스템 콜 확인
$ strace -c -p $(pgrep myapp) sleep 5
# 어떤 시스템 콜이 많은지 확인

# Step 3: 코어별 분포 확인
$ mpstat -P ALL 1 3
# 특정 코어에 부하 편중 여부

# Step 4: 어느 프로세스가 CPU를 쓰는지
$ pidstat -u 1 5
# CPU 사용률 상위 프로세스

# Step 5: wa 높음 → I/O 병목 확인
$ iostat -xz 1
# 어느 디스크에서 await, %util 높은지
```

---

## 🔬 내부 동작 원리

### CPU 시간 4가지 분류

```
커널은 각 CPU의 시간을 다음 범주로 집계:

us (user): 유저 공간에서 코드 실행 시간
  - 애플리케이션 로직 (Java bytecode, Python, C 코드)
  - JIT 컴파일된 코드 실행
  - 높으면: CPU 바운드 애플리케이션 로직
  - 해결: 알고리즘 최적화, 프로파일링

sy (system): 커널 모드에서 실행 시간
  - 시스템 콜 처리 (read, write, socket 등)
  - 인터럽트 핸들링
  - 스케줄러 실행
  - 높으면: 과도한 시스템 콜, 컨텍스트 스위칭
  - 해결: strace로 과도한 시스템 콜 탐지

wa (iowait): I/O 완료를 기다리는 시간
  - 디스크 읽기/쓰기 완료 대기
  - 네트워크 I/O 대기 (일부)
  - 높으면: 디스크 또는 네트워크 I/O 병목
  - 해결: iostat으로 디스크 확인, SSD 업그레이드

id (idle): CPU가 할 일이 없어 유휴 상태
  - 실행 가능한 태스크 없음
  - I/O 대기도 없음
  - 낮으면(0에 가까우면): CPU 포화 상태

hi (hardware interrupt): 하드웨어 인터럽트 처리 시간
  - NIC 패킷 수신, 디스크 I/O 완료 등
  - 높으면: 네트워크 패킷 폭주, IRQ 불균형

si (software interrupt): 소프트웨어 인터럽트 처리 시간
  - 네트워크 패킷 처리 (ksoftirqd)
  - 높으면: 네트워크 처리량 과다
```

### Load Average의 정확한 의미

```
Load Average = 지수 가중 이동 평균으로 계산된
               "실행 중 + 실행 대기 중 + I/O 대기 중" 태스크 수

uptime: 1분, 5분, 15분 평균 표시
load average: 8.5, 7.2, 6.1
              ↑1분  ↑5분  ↑15분

이전 레포에서 나온 개념:
  TASK_RUNNING (실행 중 or 실행 대기):    Load에 포함
  TASK_INTERRUPTIBLE (I/O 대기):         포함 안 됨
  TASK_UNINTERRUPTIBLE (I/O 대기, D상태): 포함됨!

→ wa(iowait)가 높으면 Load Average도 높아질 수 있음

4코어 서버에서 Load Average 해석:
  1.0  = 정확히 1 CPU 계속 사용 중 (여유 있음)
  4.0  = 4개 모두 바쁨 (포화 한계)
  8.0  = 4개 * 2배 = 평균적으로 4개 태스크가 대기 중
  → 응답 시간 2배 증가 예상

Load Average가 높지만 CPU id%도 높은 경우:
  I/O 대기(wa)가 Load에 포함되어 부풀려진 것
  → 실제 CPU 경쟁은 없고 디스크가 병목
```

### mpstat으로 코어별 분석

```bash
$ mpstat -P ALL 1

# 출력 예시:
11:23:45  CPU    %usr   %sys  %iowait  %irq  %soft  %idle
11:23:46  all    12.5    5.0     2.5    0.1    0.4   79.5
11:23:46    0    99.0    1.0     0.0    0.0    0.0    0.0  ← 코어 0 포화!
11:23:46    1     0.0    0.0     0.0    0.0    0.0  100.0  ← 코어 1 완전 유휴
11:23:46    2     0.0    0.0    10.0    0.0    0.0   90.0
11:23:46    3     0.0    0.0     0.0    0.1    0.4   99.5

# "전체 CPU 12.5%로 괜찮아 보이지만..."
# 코어 0이 100% = 단일 스레드 병목 (Redis 이벤트 루프, GIL 등)
# 해결: 해당 프로세스의 CPU 어피니티 확인, 멀티스레드 전환 검토

# 어느 프로세스가 코어 0을 독점하는지 확인
$ ps -eo pid,psr,pcpu,comm | awk '$3 > 50 {print}'
# psr: 현재 실행 중인 CPU 번호
```

### wa (iowait) 진단 흐름

```
wa 높음 감지
       │
       ▼
$ iostat -xz 1
  await > 10ms?  → 디스크 지연 높음
  %util > 70%?   → 디스크 포화 의심 (SSD는 다름)
  r/s vs w/s     → 읽기/쓰기 중 무엇이 원인?
       │
       ├── 특정 디스크만 높음?
       │   → 해당 디스크 사용 프로세스 확인
       │   $ iotop -bo -d 1
       │
       └── 전체 디스크 높음?
           → Page Cache 미스 (메모리 부족?)
           → 대량 쓰기 flush?
           $ cat /proc/meminfo | grep Dirty
```

---

## 💻 실전 실험

### 실험 1: CPU 각 영역 인위적으로 올리기

```bash
# us 높이기 (유저 공간 CPU 바운드)
$ stress-ng --cpu 4 --timeout 10s &
$ top  # us가 높아짐

# sy 높이기 (시스템 콜 많이 호출)
$ stress-ng --syscall 4 --timeout 10s &
$ top  # sy가 높아짐
# strace -c -p $(pgrep stress-ng) sleep 3  # 어떤 시스템 콜인지 확인

# wa 높이기 (디스크 I/O)
$ stress-ng --hdd 2 --hdd-write-size 4k --timeout 10s &
$ top  # wa가 높아짐
$ iostat -xz 1  # 디스크 현황 동시 확인

# 각 영역 비교 확인
$ mpstat 1 3
```

### 실험 2: Load Average와 실제 응답 시간 관계

```bash
# 부하 증가에 따른 응답 시간 측정
$ cat << 'EOF' > /tmp/load_test.sh
#!/bin/bash
for workers in 1 2 4 8 16; do
    # 백그라운드 부하 생성
    for i in $(seq 1 $workers); do
        stress-ng --cpu 1 --timeout 30s &
    done

    sleep 5  # 안정화 대기

    # 응답 시간 측정
    start=$(date +%s%3N)
    for i in $(seq 1 10); do
        sleep 0.001  # 경량 작업
    done
    end=$(date +%s%3N)

    echo "Workers=$workers, Load=$(uptime | awk '{print $10}'), Time=$((end-start))ms"
    kill %1 %2 %3 %4 %5 2>/dev/null
    sleep 3
done
EOF
$ bash /tmp/load_test.sh
```

### 실험 3: si (softirq) 높은 경우 — 네트워크 폭주

```bash
# 네트워크 패킷 폭주로 si 올리기
$ apt-get install -y iperf3
$ iperf3 -s &  # 서버
$ iperf3 -c localhost -P 16 -t 30s &  # 16 병렬 스트림

$ top
# si가 올라감 (ksoftirqd가 패킷 처리)

# 어느 CPU에서 softirq 처리하는지
$ mpstat -P ALL 1 | grep -v "^$"
# 특정 코어에 si 집중 → IRQ 어피니티 설정 필요

# NIC 인터럽트 분산 (RSS/RPS)
$ cat /proc/interrupts | grep eth0
# 어느 CPU가 NIC 인터럽트 처리하는지 확인
```

---

## 📊 성능/비용 비교

| CPU 지표 | 높을 때 의심 원인 | 다음 확인 단계 |
|---------|----------------|-------------|
| `us` 높음 | 애플리케이션 CPU 바운드 | `perf top`, `jstack` |
| `sy` 높음 | 과도한 시스템 콜 | `strace -c -p <pid>` |
| `wa` 높음 | 디스크/네트워크 I/O 병목 | `iostat -xz 1`, `iotop` |
| `si` 높음 | 네트워크 패킷 폭주 | `sar -n DEV 1`, `/proc/interrupts` |
| `hi` 높음 | 하드웨어 인터럽트 과다 | `/proc/interrupts`, IRQ 어피니티 |

**Load Average 해석 기준 (4코어)**:

```
Load Average  해석           권장 조치
0~4           정상           모니터링 지속
4~6           주의           병목 원인 조사
6~8           높음           즉시 조사 필요
8+            심각           조치 필요
```

---

## ⚖️ 트레이드오프

**wa(iowait)의 해석 함정**

`wa`가 0%라고 해서 I/O 병목이 없다는 뜻은 아니다. `wa`는 "CPU가 I/O를 기다리느라 유휴인 시간"이다. 이벤트 루프(epoll 기반)는 I/O를 기다리더라도 CPU를 블로킹하지 않으므로 `wa`에 잡히지 않는다. 반대로 스레드 기반 서버에서 I/O 대기가 많으면 `wa`가 높게 나온다.

**단일 코어 병목과 Load Average**

멀티코어 서버에서 단일 스레드 애플리케이션(Redis 이벤트 루프, Python GIL 등)은 코어 1개만 사용한다. 전체 CPU 사용률은 낮지만(예: 4코어에서 25%) 해당 코어는 100%다. Load Average도 약 1.0 수준으로 "정상"처럼 보인다. 하지만 이 단일 코어가 처리량의 한계가 된다. `mpstat -P ALL`로 코어별 확인이 필수다.

---

## 📌 핵심 정리

```
CPU 시간 분류:
  us: 유저 공간 실행 → 애플리케이션 로직
  sy: 커널 모드 실행 → 시스템 콜, 스케줄러
  wa: I/O 대기      → 디스크/네트워크 병목
  id: 유휴          → 0에 가까우면 포화
  si: 소프트웨어 인터럽트 → 네트워크 패킷 처리

진단 흐름:
  sy 높음 → strace -c -p <pid> → 어떤 시스템 콜?
  wa 높음 → iostat -xz 1      → 어떤 디스크?
  us 높음 → perf top           → 어떤 함수?
  si 높음 → /proc/interrupts   → NIC IRQ 분산?

Load Average:
  실행 중 + 실행 대기 + D상태 I/O 대기 태스크 수
  코어 수와 비교: 초과하면 일부 태스크가 대기 중

핵심 명령어:
  top / htop          → 전체 CPU 현황
  mpstat -P ALL 1     → 코어별 분포 (편중 확인)
  pidstat -u 1        → 프로세스별 CPU 사용률
  uptime              → Load Average (1/5/15분)
  strace -c -p <pid>  → 시스템 콜 통계
```

---

## 🤔 생각해볼 문제

**Q1.** 16코어 서버에서 Load Average가 4.0이고 `wa` 30%, `id` 55%다. 실제로 CPU가 부족한 상황인가?

<details>
<summary>해설 보기</summary>

CPU가 부족한 상황이 아니다. `id` 55%는 CPU 여유가 충분함을 의미한다. Load Average 4.0은 D상태 I/O 대기 태스크가 포함된 수치다. `wa` 30%는 16코어 중 상당수가 I/O를 기다리고 있음을 의미한다. 이 상황의 실제 병목은 CPU가 아니라 디스크다. `iostat -xz 1`로 어느 디스크의 `await`이 높은지 확인하고, `iotop`으로 어느 프로세스가 I/O를 일으키는지 확인하는 것이 다음 단계다.

</details>

**Q2.** 배치 작업 실행 중 Redis 서버의 응답 속도가 2배 느려졌다. CPU `us`는 변화 없고 `sy`가 5%에서 25%로 올랐다. 원인은?

<details>
<summary>해설 보기</summary>

배치 작업이 많은 시스템 콜을 유발해 커널 시간이 늘어난 것이다. 배치 작업이 파일 I/O, 소켓 작업, 메모리 할당 등 시스템 콜 집약적인 작업을 하면 CPU의 sy가 증가한다. 컨텍스트 스위칭도 증가했을 가능성이 있다. Redis는 단일 스레드라 컨텍스트 스위칭 직접 영향은 제한적이지만, 커널이 다른 작업 처리로 바빠지면 Redis 이벤트 루프로의 CPU 전환이 지연될 수 있다. `strace -c -p $(pgrep batch_job)` 으로 배치 작업의 시스템 콜 패턴을 확인하고, `vmstat 1`으로 `cs`(컨텍스트 스위칭) 수를 확인한다.

</details>

**Q3.** `top`의 `hi`(하드웨어 인터럽트)가 15%로 높다. 어떤 하드웨어가 원인이고 어떻게 진단하는가?

<details>
<summary>해설 보기</summary>

`/proc/interrupts`를 확인해 어떤 인터럽트 번호가 빠르게 증가하는지 본다. `watch -n 1 "cat /proc/interrupts"`로 실시간으로 어느 인터럽트가 많이 발생하는지 확인한다. 일반적으로 `hi`가 높은 원인은 NIC(고속 네트워크 패킷 수신), NVMe SSD(I/O 완료 인터럽트)다. NIC의 경우 `ethtool -l eth0`로 IRQ 큐 수를 확인하고, RSS(Receive Side Scaling)로 여러 코어에 인터럽트를 분산할 수 있다. 특정 코어에만 `hi`가 집중된다면 `mpstat -P ALL`에서 확인하고 IRQ 어피니티(`/proc/irq/<n>/smp_affinity`)를 조정한다.

</details>

---

**[홈으로 🏠](../README.md)** | **[다음: 메모리 분석 ➡️](./02-memory-analysis.md)**
