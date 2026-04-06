# 05. strace와 perf — 시스템 콜 추적과 Flame Graph

## 🎯 핵심 질문

- `strace -c -p <pid>`로 어떤 정보를 얻고 어떻게 해석하는가?
- `perf record + perf report`로 CPU 핫스팟을 어떻게 찾는가?
- Flame Graph는 무엇을 시각화하고, 어떤 문제를 발견할 수 있는가?
- Java 애플리케이션에 `perf`를 적용할 때 어떤 추가 설정이 필요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**`top`에서 CPU 30%인데 어디서 쓰는지 모른다.** `perf top`을 실행하면 실시간으로 어떤 함수가 CPU를 많이 사용하는지 볼 수 있다. 커널 함수, 라이브러리 함수, 애플리케이션 함수를 모두 볼 수 있다.

**Redis의 `sy`(시스템)가 갑자기 15%로 올랐다.** `strace -c -p $(pgrep redis-server) sleep 5`로 5초 동안 Redis가 어떤 시스템 콜을 얼마나 호출하는지 통계를 얻는다. `epoll_wait`가 아닌 다른 시스템 콜이 급증했다면 원인을 좁힐 수 있다.

**Spring Boot 서비스에서 특정 API만 느리다.** `perf record`로 해당 API 호출 중 CPU 프로파일을 수집하고, Flame Graph를 그리면 어떤 메서드 체인이 가장 많은 CPU를 소비하는지 한눈에 보인다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: strace를 운영 환경에서 과도하게 사용
$ strace -p $PID  # 모든 시스템 콜 출력 (운영 환경 40~70% 성능 저하!)
# → 올바른 방법: -c 옵션으로 통계만 수집 (부담 적음)
$ strace -c -p $PID sleep 30  # 통계만

# 실수 2: Java perf 프로파일이 "perf-<pid>.map 없음" 오류
$ perf record -F 99 -p $JAVA_PID
$ perf report
# → JVM 메서드 이름이 보이지 않음 (hex 주소만)
# → JVM이 JIT 컴파일된 코드의 심볼 맵을 생성해야 함
# $ java -XX:+PreserveFramePointer -jar app.jar  (필수!)
# → /tmp/perf-<pid>.map 자동 생성됨

# 실수 3: perf로 I/O 병목 찾으려 함
$ perf record -e block:block_rq_complete -a  # 블록 I/O 이벤트
# → 이건 맞지만 일반 CPU 프로파일로는 I/O 대기 안 보임
# → I/O 병목은 iostat + iotop으로 찾고, perf는 CPU 분석에 사용
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 도구별 용도 구분
#
# strace: "무슨 시스템 콜을 얼마나?"
#   → sy 높을 때, 시스템 콜 패턴 확인
#
# perf:   "어떤 함수에서 CPU를 얼마나?"
#   → us 높을 때, CPU 핫스팟 확인
#
# Flame Graph: "전체 호출 스택에서 CPU 분포"
#   → 복잡한 코드베이스에서 핫스팟 시각화

# 종합 진단 흐름
$ top  # us 높음 → perf, sy 높음 → strace

# CPU 핫스팟 (us 높음):
$ perf top -p $PID  # 실시간 핫스팟
$ perf record -F 99 -g -p $PID sleep 30  # 30초 수집
$ perf report --stdio | head -50  # 결과 확인
$ perf script | flamegraph.pl > flame.svg  # Flame Graph

# 시스템 콜 분석 (sy 높음):
$ strace -c -p $PID sleep 30  # 30초 통계
$ strace -e trace=read,write,epoll_wait -T -p $PID  # 특정 콜 타이밍
```

---

## 🔬 내부 동작 원리

### strace 동작 원리

```
strace 동작 방식 (ptrace 시스템 콜 기반):

strace → ptrace(PTRACE_ATTACH, target_pid)
          ↓
target_pid: 매 시스템 콜 진입/종료 시 일시 정지
          ↓
strace:  시스템 콜 번호, 인자, 반환값 읽기 + 출력
          ↓
strace → ptrace(PTRACE_CONT) → target_pid 재개

성능 영향:
  모든 시스템 콜마다 strace 프로세스가 개입
  → 40~70% 처리량 감소
  → -c 옵션: 통계만 (상대적으로 가벼움, ~20~30% 저하)
  → 운영 환경: -c 만 사용, 또는 perf trace 사용 (~5% 저하)

strace -c 출력 해석:
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- --------
 45.23    0.023456          23      1024           epoll_wait  ← 이벤트 루프
 30.12    0.015634          15      1042           read        ← 데이터 읽기
 15.45    0.008012           8      1001           write       ← 데이터 쓰기
  5.20    0.002700           2      1350           recvfrom    ← 소켓 수신
  2.10    0.001100           1      1100           sendto      ← 소켓 송신

해석:
  epoll_wait 45%: 이벤트 루프가 대부분 대기 (정상)
  read 30%:       데이터 읽기 (소켓, 파일)
  futex 높음:     락 경합 (멀티스레드 동기화 문제)
  poll/select:    Old-style I/O 감시 (epoll이 아닌 경우)
  lseek + read:   랜덤 파일 I/O 패턴
```

### perf 동작 원리

```
perf 동작 방식 (하드웨어 성능 카운터):

1. PMU (Performance Monitoring Unit) 설정
   → CPU가 지원하는 하드웨어 카운터 사용
   → 예: cpu-cycles, cache-misses, branch-misses

2. 샘플링 기반 프로파일
   perf record -F 99 -g:
     F 99: 초당 99회 샘플링 (균등 분포)
     g:    콜스택 캡처 (Frame Pointer 필요)

3. 인터럽트 기반 샘플링
   N 이벤트마다 CPU 인터럽트 발생
   → 현재 실행 중인 명령어 주소 기록
   → 콜스택 역추적 (unwind)

4. 결과 분석
   perf report: 함수별 CPU 점유율
   perf script: 샘플 상세 목록 (Flame Graph용)

성능 영향:
  F=99 (100Hz 샘플링): ~1~5% 오버헤드
  F=9999 (10KHz): ~10% 오버헤드
  운영 환경: F=49 또는 F=99 권장
```

### Flame Graph 읽는 방법

```
Flame Graph 구조:

  ┌────────────────────────────────────────────────────────┐
  │                        main()                          │ ← 너비 = CPU 시간 비율
  ├──────────────────────────┬─────────────────────────────┤
  │        handleRequest()   │    backgroundTask()         │
  ├──────────────┬───────────┤                             │
  │  queryDB()   │serialize()│                             │
  ├──────────────┤           │                             │
  │ jdbc.query() │           │                             │
  └──────────────┴───────────┴─────────────────────────────┘

  X축 = CPU 시간 비율 (넓을수록 CPU 더 사용)
  Y축 = 콜스택 깊이 (위가 현재 실행 함수)
  색상 = 의미 없음 (구별용)

읽는 방법:
  1. 가장 넓은 타워를 찾음 → CPU 핫스팟
  2. 해당 타워의 맨 위 함수 → 직접 CPU 소비 함수
  3. 아래로 내려가며 → 호출 경로 확인

발견할 수 있는 문제:
  ① 넓고 평평한 타워: 하나의 함수가 CPU를 많이 씀
  ② 매우 깊은 재귀: 스택 오버플로 위험
  ③ 예상치 못한 라이브러리 함수: GC, 직렬화 오버헤드
  ④ 커널 함수 비중: 시스템 콜 과다
```

### Java에서 perf 사용하기

```bash
# Java perf 프로파일 설정

# 필수: Frame Pointer 보존 (JVM JIT 코드 콜스택 추적)
$ java -XX:+PreserveFramePointer -jar app.jar &
PID=$!

# 선택 1: JVM 심볼 맵 생성 (JIT 메서드 이름 표시)
# Java 8-16:
$ java -XX:+PreserveFramePointer \
       -XX:+UnlockDiagnosticVMOptions \
       -XX:+DebugNonSafepoints \
       -jar app.jar &
# → /tmp/perf-<pid>.map 자동 생성

# Java 17+: JFR (Java Flight Recorder) 통합
$ java -XX:StartFlightRecording=duration=60s,filename=profile.jfr \
       -jar app.jar

# perf 데이터 수집
$ perf record -F 99 -g -p $PID sleep 30
$ perf report --stdio --no-children | head -50

# Flame Graph 생성
$ git clone https://github.com/brendangregg/FlameGraph
$ perf script | FlameGraph/stackcollapse-perf.pl | \
  FlameGraph/flamegraph.pl > java_flame.svg

# 출력에서 볼 수 있는 것:
# [java] Interpreter_loop  ← 인터프리터 (JIT 전)
# com/example/Service.process  ← JIT 컴파일된 메서드
# GC 관련 함수가 넓으면 → GC 튜닝 필요
# sun/misc/Unsafe.* 넓으면 → Direct Memory 작업
```

---

## 💻 실전 실험

### 실험 1: strace로 시스템 콜 병목 찾기

```bash
# Redis 실행 중 시스템 콜 패턴 분석
$ redis-cli CONFIG SET hz 100  # 높은 타이머 설정
$ redis-benchmark -n 100000 -c 50 &  # 부하 생성

# 30초 동안 Redis 시스템 콜 통계
$ strace -c -p $(pgrep redis-server) sleep 30 2>&1

# 정상 출력 예:
# epoll_wait: 65% (이벤트 루프 대기)
# read: 20% (데이터 읽기)
# write: 10% (응답 쓰기)

# hz=100으로 타이머 콜 증가 시:
# nanosleep/clock_gettime이 높아질 수 있음
# → 불필요한 타이머가 시스템 콜 유발

# 특정 콜 시간 측정
$ strace -e trace=epoll_wait -T -p $(pgrep redis-server) 2>&1 | head -10
# epoll_wait(...) = 5  <0.001234>  ← 1.234ms 소요
```

### 실험 2: perf로 CPU 핫스팟 찾기

```bash
# CPU 집약 작업 실행
$ cat > /tmp/cpu_heavy.py << 'EOF'
import hashlib, time

def hash_data(data):
    for _ in range(1000):
        hashlib.sha256(data).hexdigest()

while True:
    hash_data(b"test data " * 100)
EOF
$ python3 /tmp/cpu_heavy.py &
PID=$!

# perf로 30초 수집
$ perf record -F 99 -g -p $PID sleep 30

# 결과 확인
$ perf report --stdio --no-children | head -30

# 예상 출력:
# 85.23%  python3  libssl.so  [.] SHA256_Update
# 10.12%  python3  python3    [.] _PyObject_GenericGetAttrWithDict
# ...

kill $PID
```

### 실험 3: Java Flame Graph 생성

```bash
# Java 앱 실행 (심볼 맵 생성 옵션 포함)
$ java -XX:+PreserveFramePointer \
       -XX:+UnlockDiagnosticVMOptions \
       -XX:+DebugNonSafepoints \
       -Xmx512m \
       -jar /path/to/app.jar &
PID=$!

sleep 5  # 시작 대기

# 부하 생성
$ ab -n 10000 -c 20 http://localhost:8080/api/endpoint &

# perf 데이터 수집 (30초)
$ perf record -F 99 -g -p $PID sleep 30

# FlameGraph 도구 설치
$ git clone --depth=1 https://github.com/brendangregg/FlameGraph /tmp/FlameGraph

# Flame Graph 생성
$ perf script | /tmp/FlameGraph/stackcollapse-perf.pl | \
  /tmp/FlameGraph/flamegraph.pl \
  --title "Java App CPU Profile" \
  --width 1800 > /tmp/java_flame.svg

echo "Flame Graph 생성 완료: /tmp/java_flame.svg"
# 브라우저에서 열어 핫스팟 확인

# GC 시간 확인 (Flame Graph에서 너무 넓으면 문제)
$ perf script | grep "G1\|Concurrent\|Parallel" | wc -l
```

---

## 📊 성능/비용 비교

**진단 도구별 오버헤드와 정밀도**:

```
도구              오버헤드    실시간  정밀도    적합한 상황
strace (전체)     40~70%     가능    시스템콜  개발/테스트 환경만
strace -c         20~30%     가능    시스템콜  sy 높을 때 통계
perf top          1~5%       가능    CPU 함수  실시간 핫스팟 확인
perf record       1~5%       불가    CPU 함수  정밀 프로파일
perf trace        5~10%      가능    시스템콜  운영 환경 strace 대체
eBPF/bpftrace     1~3%       가능    다양      운영 환경 최적
JFR (Java)        1~2%       가능    JVM 내부  Java 운영 환경
```

**strace 결과 해석 가이드**:

```
시스템 콜        높을 때 의미           다음 확인
epoll_wait       정상 (이벤트 루프)     비율이 너무 낮으면 다른 콜이 높음
futex            락 경합               멀티스레드 동기화 문제
read/write       I/O 집약              버퍼 크기, Page Cache 확인
mmap/munmap      메모리 할당/해제 빈번  할당기 최적화, 큰 버퍼 재사용
brk              Heap 확장             메모리 할당 패턴 최적화
clone            스레드 생성 빈번       스레드 풀 도입
openat/close     파일 열고 닫기 반복   파일 핸들 캐싱
```

---

## ⚖️ 트레이드오프

**strace vs perf trace vs eBPF**

`strace`는 `ptrace` 기반으로 가장 상세하지만 오버헤드가 크다. `perf trace`는 `perf_event` 기반으로 오버헤드가 낮고 운영 환경에서 더 안전하다. `bpftrace`는 eBPF 기반으로 가장 낮은 오버헤드에 커스텀 분석이 가능하다. 운영 환경에서 분석이 필요하다면 `perf trace` 또는 `bpftrace`를 우선 고려한다.

**Flame Graph 샘플링 주기**

높은 주기(예: 9999Hz)는 더 정밀한 프로파일을 제공하지만 오버헤드가 증가한다. 낮은 주기(예: 49Hz)는 오버헤드가 적지만 짧은 함수는 포착하지 못한다. 99Hz는 대부분의 경우에 좋은 균형이다. 짧은 레이턴시를 유발하는 함수(수 μs)를 찾으려면 더 높은 주기가 필요하다.

---

## 📌 핵심 정리

```
strace 용도:
  sy(시스템) CPU 높을 때 → 어떤 시스템 콜이 많은지
  -c: 통계만 수집 (오버헤드 ~20~30%)
  -T: 각 콜 소요 시간
  -e trace=<syscall>: 특정 콜만 추적

perf 용도:
  us(유저) CPU 높을 때 → 어떤 함수가 핫스팟인지
  perf top: 실시간 핫스팟
  perf record -F 99 -g: 샘플 수집
  perf report: 함수별 CPU 점유율
  perf script | flamegraph.pl: Flame Graph

Java perf 설정:
  -XX:+PreserveFramePointer (콜스택 추적 필수)
  -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints
  → /tmp/perf-<pid>.map 자동 생성 (JIT 심볼)

Flame Graph 해석:
  넓은 타워 = CPU 많이 사용
  맨 위 함수 = 직접 CPU 소비
  아래 = 호출 경로

운영 환경 안전한 대안:
  strace -c (통계만)
  perf record -F 49 (낮은 주기)
  perf trace (perf 기반 시스템 콜)
  bpftrace (eBPF, 최저 오버헤드)
```

---

## 🤔 생각해볼 문제

**Q1.** `strace -c`로 Redis를 분석했더니 `futex` 시스템 콜이 전체의 40%를 차지한다. 이것이 의미하는 바는?

<details>
<summary>해설 보기</summary>

Redis는 단일 스레드 이벤트 루프인데 `futex`(Fast Userspace Mutex)가 많다는 것은 이상하다. 두 가지 가능성이 있다. 첫째, Redis 6.0+의 Threaded I/O가 활성화된 경우(`io-threads > 1`) I/O 스레드와 메인 스레드 간 동기화에 `futex`가 사용된다. 둘째, jemalloc 같은 메모리 할당기 내부에서 arena 간 동기화에 `futex`를 사용하는 경우다. `io-threads` 설정과 `CONFIG GET io-threads`로 확인하고, I/O 스레드가 활성화됐다면 이것은 정상이다.

</details>

**Q2.** Flame Graph에서 가장 넓은 타워가 GC 관련 함수(`G1_Concurrent_Marking_Task`)이고 CPU의 25%를 차지한다. 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

GC가 전체 CPU의 25%를 사용한다는 것은 과도한 수준이다. 다음 단계로 진단한다. `jstat -gcutil <pid> 1`으로 GC 빈도와 지속 시간을 확인한다. 주요 원인과 해결: Old Generation이 지속 차오르면 Heap 누수나 메모리 부족이다(`-Xmx` 증가 또는 객체 참조 정리). Young Generation GC가 너무 빈번하면 짧은 수명의 객체를 대량 생성하는 것이다(객체 재사용, StringBuilder 활용). `jmap -histo <pid>`로 가장 많은 메모리를 차지하는 클래스를 확인하고 코드를 개선한다.

</details>

**Q3.** `perf record` 결과에서 함수 이름 대신 `[unknown]`이 많이 보인다. 어떤 원인이고 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

세 가지 원인이 있다. 첫째, JIT 컴파일된 코드(Java, Node.js V8 등)는 동적으로 생성되어 심볼이 없다. Java는 `-XX:+PreserveFramePointer`와 `/tmp/perf-<pid>.map`으로 해결한다. Node.js는 `--perf-basic-prof` 플래그로 해결한다. 둘째, 디버그 심볼이 없는 바이너리다. `apt-get install -y <package>-dbgsym` 또는 소스 컴파일 시 `-g` 플래그로 해결한다. 셋째, Frame Pointer가 없는 코드다(`-fomit-frame-pointer` 컴파일 옵션). `--call-graph dwarf` 옵션으로 DWARF 언와인딩을 사용하면 Frame Pointer 없이도 콜스택을 추적할 수 있다(오버헤드 증가).

</details>

---

**[⬅️ 이전: 네트워크 분석](./04-network-analysis.md)** | **[홈으로 🏠](../README.md)**
