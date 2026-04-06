# 04. OOM in Containers — JVM과 cgroups의 충돌

## 🎯 핵심 질문

- Java 8 JVM이 컨테이너 메모리 한도를 무시하고 호스트 메모리를 참조하는 이유는?
- `--memory=512m`인데 JVM `-Xmx1g`를 설정하면 정확히 어떤 순서로 OOM이 발생하는가?
- `-XX:+UseContainerSupport`가 내부적으로 어떤 값을 읽어 Heap 크기를 결정하는가?
- `-XX:MaxRAMPercentage`의 75%는 어떤 근거로 권장되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**"Java 프로세스가 설명 없이 종료됐다."** exit code 137, OOMKilled=true인데 JVM OOM 에러도 없다. 커널 OOM Killer가 JVM 프로세스를 강제 종료한 것이다. JVM이 cgroups 한도를 인식하지 못해 과도하게 메모리를 사용했기 때문이다.

**Java 8 서버를 컨테이너로 마이그레이션했더니 메모리 관련 이슈가 발생한다.** Java 8은 `/proc/meminfo`로 호스트 전체 메모리를 읽어 Heap 크기를 계산한다. 호스트 메모리가 64GB라면 Heap이 16GB로 설정되지만 컨테이너 한도는 2GB — OOM이 발생할 수밖에 없다.

**`-Xmx`를 설정했는데도 OOM이 발생한다.** Heap은 `-Xmx`로 제한되지만 Metaspace, Direct Buffer, JIT 코드 캐시, 스레드 스택은 Native Memory로 추가 사용된다. 컨테이너 한도 = `-Xmx` 값으로 설정하면 필연적으로 OOM이 발생한다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: Java 8에서 컨테이너 메모리 한도 무시
$ docker run --memory=2g openjdk:8 java -XX:+PrintFlagsFinal -version \
  2>&1 | grep MaxHeapSize
MaxHeapSize = 16106127360  ← 15GB! (호스트 64GB의 1/4)
# 컨테이너 한도 2GB인데 JVM이 15GB Heap을 잡으려 함
# → 즉시 OOM Kill

# 실수 2: -Xmx = 컨테이너 한도로 설정
$ docker run --memory=2g myapp java -Xmx2g -jar app.jar
# JVM 전체 메모리:
#   Heap:         2GB (-Xmx)
#   Metaspace:    ~300MB
#   Thread Stack: ~200MB (200threads * 1MB)
#   Code Cache:   ~256MB
#   Direct Buffer: ~100MB
# 합계: ~2.9GB > 컨테이너 한도 2GB → OOM Kill

# 실수 3: OOMKilled를 애플리케이션 버그로 오인
$ kubectl get pod myapp-xxx -o yaml | grep -A 5 "lastState"
lastState:
  terminated:
    exitCode: 137
    reason: OOMKilled  ← 명확히 표시됨
# 이것을 애플리케이션 로그에서 찾으려 함 (없음, 커널이 강제 종료)
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# Java 11+: UseContainerSupport로 cgroups 인식
$ docker run --memory=2g openjdk:11 java \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+PrintFlagsFinal -version 2>&1 | grep MaxHeapSize
MaxHeapSize = 1610612736  ← 1.5GB (2GB * 75%)

# JVM 메모리 계획 (컨테이너 2GB 기준)
# Heap:         1.5GB (MaxRAMPercentage=75%)
# Metaspace:    300MB  (남은 500MB에서)
# Thread Stack: 100MB
# Code Cache:   100MB
# 합계:         ~2.0GB ← 컨테이너 한도 내

# Kubernetes Deployment 설정 예시
cat << 'EOF'
resources:
  requests:
    memory: "2Gi"
  limits:
    memory: "2Gi"
env:
  - name: JAVA_OPTS
    value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0
            -XX:MaxMetaspaceSize=256m -Xss512k
            -XX:+ExitOnOutOfMemoryError"
EOF

# 실제 사용량 모니터링
$ jcmd $(pgrep java) VM.native_memory summary
Total: reserved=2048MB, committed=1850MB
- Java Heap:  reserved=1536MB, committed=1536MB
- Class:      reserved=300MB, committed=50MB
- Thread:     reserved=100MB, committed=100MB
```

---

## 🔬 내부 동작 원리

### Java 8의 메모리 인식 문제

```
Java 8의 Heap 크기 자동 계산:

1. /proc/meminfo 읽기
   MemTotal: 67108864 kB  ← 호스트 전체 64GB 메모리

2. Heap 크기 = MemTotal / 4 (기본값 -XX:MaxRAMFraction=4)
   = 64GB / 4 = 16GB

3. cgroups 메모리 한도? 무시!
   /sys/fs/cgroup/memory/.../memory.limit_in_bytes → 읽지 않음

컨테이너 --memory=2g로 실행 시:
  JVM이 계산한 Heap: 16GB
  cgroups 한도:       2GB
  → JVM이 16GB 메모리 요청 시 cgroups가 거부
  → OOM Kill (JVM OOMError 아님)

문제: cgroups v2 경로도 Java 8은 모름
  /sys/fs/cgroup/memory.max (v2) → 읽지 않음
  /sys/fs/cgroup/memory/memory.limit_in_bytes (v1) → 읽지 않음
```

### Java 10+ UseContainerSupport 동작

```
Java 10에서 JEP 307으로 도입:
  -XX:+UseContainerSupport (Java 11+에서 기본 활성화)

cgroups 메모리 읽기 순서:
  1. cgroups v2: /sys/fs/cgroup/memory.max
  2. cgroups v1: /sys/fs/cgroup/memory/memory.limit_in_bytes
  3. 없으면: /proc/meminfo (호스트 메모리)

읽어온 컨테이너 메모리 한도로 Heap 계산:
  -XX:MaxRAMPercentage=75.0 → Heap = 컨테이너메모리 * 0.75
  -XX:InitialRAMPercentage=50.0 → 초기 Heap = 컨테이너메모리 * 0.50

예시 (컨테이너 --memory=4g):
  컨테이너 한도: 4GB = 4294967296 bytes
  MaxHeapSize = 4GB * 75% = 3GB
  InitialHeapSize = 4GB * 50% = 2GB

Java 버전별 UseContainerSupport 기본값:
  Java 8  (8u191 이전): 없음 (수동 설정 필요)
  Java 8  (8u191 이후): 있음 (수동 활성화 필요)
  Java 11+:             기본 활성화
  Java 17+:             기본 활성화 + 개선됨
```

### JVM 전체 메모리 구성

```
컨테이너 --memory=2g 기준 JVM 메모리 분해:

구역              크기(예시)  JVM 옵션                  관리 주체
─────────────────────────────────────────────────────────────
Java Heap          1.5GB    -Xmx 또는 MaxRAMPercentage  GC
Metaspace           256MB   -XX:MaxMetaspaceSize         JVM
Thread Stack        100MB   -Xss (스레드 수 * 스택크기)  OS
Code Cache          128MB   -XX:ReservedCodeCacheSize    JIT
Direct Buffer       64MB    -XX:MaxDirectMemorySize      NIO
GC 오버헤드         ~50MB   GC 종류에 따라 다름          GC
JVM 내부 (심볼 등)   ~50MB   없음 (고정)                 JVM
─────────────────────────────────────────────────────────────
합계               ~2.1GB   ← 2GB 한도 초과 위험!

안전한 설정 (컨테이너 2GB):
  MaxRAMPercentage=70% → Heap 1.4GB
  MaxMetaspaceSize=256m
  Xss=512k (200 스레드 * 0.5MB = 100MB)
  ReservedCodeCacheSize=128m
  MaxDirectMemorySize=64m
  합계 ≈ 1.95GB → 안전 마진 2.5%

권장 메모리 계획:
  컨테이너 한도 = Heap / 0.70
  Heap 1.4GB가 필요 → 컨테이너 2GB
  Heap 3GB가 필요 → 컨테이너 4.3GB
```

### cgroups OOM Killer 발동 순서

```
컨테이너 메모리 한도 초과 시 커널 동작:

[JVM이 malloc() 또는 mmap() 호출]
         │
         ▼
[cgroups 메모리 계정 확인]
  현재 사용량 >= memory.max?
         │
    아니오 │ 예
         │   │
         │   ▼
         │ [페이지 회수 시도]
         │   page cache eviction
         │   anonymous page reclaim
         │   │
         │   회수 성공   │ 회수 실패
         │             │
         ▼             ▼
   [할당 성공]   [cgroups OOM Kill]
                  ↓
   해당 cgroup에서 oom_score 가장 높은 프로세스 선택
   → SIGKILL 전송 (핸들러 없음, 즉시 종료)
   → 보통 메모리 가장 많이 쓰는 프로세스 (JVM)
         │
         ▼
   JVM 프로세스 종료 (exit code 137)
   → 컨테이너 PID 1 종료 → 컨테이너 종료
   → Kubernetes: OOMKilled 이유로 Pod 재시작

dmesg 로그:
Memory cgroup out of memory: Killed process 1234 (java) ...
oom_score_adj 0, oom_score 1000, oom_kill 1
Killed process 1234 (java) total-vm:3145728kB, ...
```

---

## 💻 실전 실험

### 실험 1: Java 버전별 컨테이너 메모리 인식 비교

```bash
# Java 8 (컨테이너 인식 없음)
$ docker run --rm --memory=512m openjdk:8 \
  java -XX:+PrintFlagsFinal -version 2>&1 | grep MaxHeapSize
#  uintx MaxHeapSize = XXXX  ← 호스트 메모리 기준

# Java 17 (UseContainerSupport 기본 활성화)
$ docker run --rm --memory=512m openjdk:17 \
  java -XX:+PrintFlagsFinal -version 2>&1 | grep MaxHeapSize
#  size_t MaxHeapSize = 134217728  ← 128MB (512MB * 25% 기본값)

# MaxRAMPercentage 설정
$ docker run --rm --memory=512m openjdk:17 \
  java -XX:MaxRAMPercentage=75.0 -XX:+PrintFlagsFinal -version 2>&1 \
  | grep -E "MaxHeapSize|MaxRAMPercentage"
#  double MaxRAMPercentage = 75.000000
#  size_t MaxHeapSize = 402653184  ← 384MB (512MB * 75%)
```

### 실험 2: OOM Kill 재현 및 진단

```bash
# 컨테이너 한도보다 큰 Heap 설정 → OOM Kill 유발
$ docker run --rm --memory=512m --name oom-jvm openjdk:17 \
  java -Xmx512m -jar /app.jar
# JVM Heap 512MB + Native ~300MB = 800MB > 512MB → OOM Kill

# 올바른 설정으로 수정
$ docker run --rm --memory=512m --name oom-jvm openjdk:17 \
  java -XX:+UseContainerSupport \
       -XX:MaxRAMPercentage=70.0 \
       -XX:MaxMetaspaceSize=128m \
       -Xss256k \
       -jar /app.jar
# Heap: 358MB, Metaspace: 128MB, Stack: ~50MB
# 합계: ~540MB... 아직 위험

# 더 보수적인 설정
$ docker run --rm --memory=512m --name oom-jvm openjdk:17 \
  java -XX:+UseContainerSupport \
       -XX:MaxRAMPercentage=50.0 \   # 50%만 Heap으로
       -XX:MaxMetaspaceSize=96m \
       -Xss256k \
       -jar /app.jar
# Heap: 256MB + 나머지 256MB에서 Native 사용

# OOM 발생 시 즉시 Heap Dump
$ java -XX:+HeapDumpOnOutOfMemoryError \
       -XX:HeapDumpPath=/tmp/heapdump.hprof \
       -XX:+ExitOnOutOfMemoryError \
       -jar /app.jar
```

### 실험 3: 실제 JVM 메모리 사용 현황 측정

```bash
# NMT로 JVM 메모리 영역별 분석
$ docker run -d --name jvm-test \
  --memory=1g \
  openjdk:17 java \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=70.0 \
  -XX:NativeMemoryTracking=summary \
  -XX:+UnlockDiagnosticVMOptions \
  -jar /app.jar

$ PID=$(docker inspect --format '{{.State.Pid}}' jvm-test)
$ nsenter --target $PID --pid -- jcmd 1 VM.native_memory summary
Total: reserved=1024MB, committed=876MB
- Java Heap   (reserved=716MB, committed=716MB)
- Class       (reserved=96MB,  committed=18MB)   ← Metaspace
- Thread      (reserved=52MB,  committed=52MB)
- Code        (reserved=128MB, committed=28MB)
- GC          (reserved=21MB,  committed=21MB)
- Internal    (reserved=11MB,  committed=11MB)
```

---

## 📊 성능/비용 비교

**JVM 메모리 설정 시나리오 비교 (컨테이너 2GB)**:

```
설정 방식                  Heap    총 사용  OOM 위험
──────────────────────────────────────────────────
-Xmx2g (잘못됨)            2GB    ~2.8GB  높음 ★★★
MaxRAMPercentage=75%        1.5GB  ~2.0GB  낮음 ★
MaxRAMPercentage=70%        1.4GB  ~1.9GB  매우낮음
-Xmx512m (보수적)           512MB  ~900MB  매우낮음
```

**Java 버전별 컨테이너 지원**:

```
Java 8  (8u191 미만): cgroups 인식 없음, 반드시 -Xmx 수동 설정
Java 8  (8u191 이상): -XX:+UseContainerSupport 추가 필요
Java 10-10:          UseContainerSupport 실험적
Java 11-16:          UseContainerSupport 기본 활성화
Java 17+:            UseContainerSupport + cgroups v2 지원 개선
Java 21+:            Virtual Thread와 컨테이너 최적화 강화

최신 LTS 사용 권장:
  컨테이너 환경에서 Java 17 또는 21 LTS 사용이 안전
  Java 8 사용 불가피하면: 8u191+ + UseContainerSupport 필수
```

---

## ⚖️ 트레이드오프

**MaxRAMPercentage 값 선택**

75%를 권장하는 이유는 나머지 25%를 JVM Native Memory(Metaspace, 스레드 스택, Code Cache, Direct Buffer)와 OS 오버헤드를 위해 남기기 위함이다. 애플리케이션이 많은 스레드를 사용하거나 대용량 Direct Buffer를 쓴다면 70% 이하가 안전하다. 단순한 배치 애플리케이션은 80%까지 올릴 수 있다.

**-XX:+ExitOnOutOfMemoryError의 의미**

JVM OOM Error 발생 시 프로세스를 즉시 종료하는 옵션이다. 설정하지 않으면 OOM Error를 잡아서 처리하거나 무시할 수 있는데, 이 경우 JVM이 반쪽 상태로 계속 실행될 수 있다. 컨테이너 환경에서는 OOM 발생 시 즉시 종료하고 Kubernetes가 재시작하게 하는 것이 더 안전한 패턴이다.

---

## 📌 핵심 정리

```
문제: Java 8 JVM이 /proc/meminfo(호스트 메모리)로 Heap 계산
해결: UseContainerSupport로 cgroups 메모리 한도 인식

Java 버전별:
  8u191+: -XX:+UseContainerSupport (명시적 활성화)
  11+:    기본 활성화
  17+:    cgroups v2 지원 강화

Heap 크기 설정:
  -XX:MaxRAMPercentage=75.0  (컨테이너 메모리의 75%)
  또는 명시적: -Xmx (컨테이너한도 * 0.7 기준)

JVM 전체 메모리 = Heap + Metaspace + Stack + Code Cache + ...
컨테이너 한도 = -Xmx 값 X (Native 메모리 25~30% 추가 필요)

OOM Kill 진단:
  exit code 137            → SIGKILL (커널 OOM)
  OOMKilled: true          → kubectl describe 또는 inspect
  dmesg | grep oom         → 커널 OOM 로그
  JVM 로그에 OOM 없음      → 커널 OOM (JVM 개입 없음)

권장 JVM 옵션 (컨테이너):
  -XX:+UseContainerSupport
  -XX:MaxRAMPercentage=70.0
  -XX:MaxMetaspaceSize=256m
  -Xss512k
  -XX:+ExitOnOutOfMemoryError
  -XX:+HeapDumpOnOutOfMemoryError
```

---

## 🤔 생각해볼 문제

**Q1.** Java 17에서 `-XX:+UseContainerSupport`가 기본 활성화돼 있는데, `-Xmx4g`를 직접 설정하면 `MaxRAMPercentage`보다 우선하는가?

<details>
<summary>해설 보기</summary>

그렇다. `-Xmx`는 `MaxRAMPercentage`보다 높은 우선순위를 가진다. 명시적으로 `-Xmx4g`를 설정하면 컨테이너 한도와 무관하게 Heap이 4GB로 설정된다. 컨테이너 한도가 2GB라면 이것이 OOM의 원인이 된다. 권장 방식은 `MaxRAMPercentage`만 설정하고 `-Xmx`는 명시하지 않는 것이다. 두 옵션을 동시에 쓰면 `-Xmx`가 항상 이긴다.

</details>

**Q2.** Kubernetes에서 `resources.requests.memory`와 `resources.limits.memory`를 다르게 설정할 수 있다. JVM의 `MaxRAMPercentage`는 어느 값을 기준으로 계산하는가?

<details>
<summary>해설 보기</summary>

`limits.memory`를 기준으로 계산한다. cgroups의 `memory.max`에는 limits 값이 설정된다. requests는 스케줄링에만 사용되고 cgroups에는 반영되지 않는다. 따라서 `MaxRAMPercentage`는 limits 값의 퍼센트로 Heap을 결정한다. requests와 limits가 다르면(Burstable QoS) JVM이 limits 기준으로 Heap을 크게 잡을 수 있다. 안정적인 운영을 위해 `requests = limits`(Guaranteed QoS)를 권장하는 이유다.

</details>

**Q3.** `G1GC`와 `ZGC` 중 컨테이너 환경에서 어떤 GC가 더 적합하고, 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

일반적으로 **ZGC** (또는 Shenandoah GC)가 컨테이너 환경에 더 유리하다. ZGC는 STW(Stop-The-World) 시간을 수 밀리초 이내로 유지한다. 컨테이너는 메모리 한도에 가까워지면 `memory.high` 초과로 Page Cache 회수와 스로틀링이 발생해 GC 압박이 커질 수 있다. 이 상황에서 G1GC는 Full GC(수 초 STW)가 발생할 수 있지만, ZGC는 짧은 STW를 유지한다. 단, ZGC는 CPU 오버헤드가 더 크므로 CPU 제한이 작은 컨테이너에서는 트레이드오프가 있다.

</details>

---

**[⬅️ 이전: 컨테이너 네트워킹](./03-container-networking.md)** | **[홈으로 🏠](../README.md)** | **[다음: seccomp와 capabilities ➡️](./05-seccomp-capabilities.md)**
