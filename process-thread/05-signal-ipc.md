# 05. 시그널과 IPC — SIGKILL vs SIGTERM

## 🎯 핵심 질문

- `SIGKILL`과 `SIGTERM`은 커널 레벨에서 어떻게 처리가 다른가?
- Graceful Shutdown이 `SIGTERM`에 의존하는 이유는?
- Pipe, FIFO, Message Queue, Shared Memory는 각각 무엇을 위해 설계됐는가?
- Kubernetes `terminationGracePeriodSeconds` 설정은 어떤 원리로 동작하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**배포 중 진행 중인 요청이 끊기는 문제.** 컨테이너를 종료할 때 애플리케이션이 `SIGTERM`을 받고 Graceful Shutdown을 실행해야 한다. 하지만 JVM이 `SIGTERM` 핸들러를 등록하지 않았거나, Kubernetes `preStop` 훅이 없으면 진행 중인 요청이 강제 종료된다.

**`systemctl stop`이 느리고 30초 후 강제 종료된다.** systemd는 `SIGTERM` 후 `TimeoutStopSec`(기본 90초) 안에 종료되지 않으면 `SIGKILL`을 보낸다. 애플리케이션이 `SIGTERM`을 무시하거나 핸들러가 너무 오래 걸리면 이 상황이 발생한다.

**두 프로세스 사이에서 대용량 데이터를 교환해야 한다.** 파이프는 커널 버퍼가 64KB밖에 안 된다. 대용량 데이터 전송에는 공유 메모리가 적합하다. IPC 메커니즘의 특성을 모르면 병목을 만든다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: Graceful Shutdown 없이 배포
$ kubectl delete pod my-app-xxx
# → Pod가 즉시 SIGTERM 수신
# → 애플리케이션이 SIGTERM 핸들러 없으면 바로 종료
# → 처리 중이던 요청 모두 502 에러

# 실수 2: SIGTERM으로 종료 안 될 때 바로 kill -9
$ kill -9 $PID
# → Graceful Shutdown 기회 없이 즉시 종료
# → DB 연결, 파일 핸들 등 클린업 안 됨
# → 데이터 손상 가능성

# 실수 3: Docker에서 PID 1 문제
# Dockerfile: CMD ["java", "-jar", "app.jar"]
# → java 프로세스가 PID 1이 됨
# → PID 1은 SIGTERM 기본 핸들러 없음
# → docker stop이 10초 후 SIGKILL로 강제 종료
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# SIGTERM을 정상 처리하도록 확인
$ kill -0 $PID      # 프로세스 존재 확인 (실제 시그널 전송 안 함)
$ kill -TERM $PID   # SIGTERM 전송
$ sleep 10
$ kill -0 $PID 2>/dev/null && echo "아직 살아있음" || echo "정상 종료됨"

# 프로세스가 등록한 시그널 핸들러 확인
$ cat /proc/$PID/status | grep SigCgt
SigCgt: 0000000180014203  ← 비트마스크로 캐치한 시그널 목록
# SIGTERM (15번) = 비트 14
# 0x4000 = 비트 14 = SIGTERM 캐치 여부 확인

# Docker에서 SIGTERM 정상 수신을 위한 설정
# Dockerfile (올바른 방식):
# ENTRYPOINT ["java", "-jar", "app.jar"]   ← exec 형식 (PID 1이 java)
# 또는
# ENTRYPOINT ["/tini", "--", "java", "-jar", "app.jar"]  ← tini init 사용
```

```java
// Spring Boot Graceful Shutdown 설정
// application.properties:
// server.shutdown=graceful
// spring.lifecycle.timeout-per-shutdown-phase=30s

// 직접 ShutdownHook 등록 (필요 시)
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    log.info("SIGTERM 수신 - Graceful Shutdown 시작");
    // DB 연결 정리, 진행 중 작업 완료 대기 등
}));
```

---

## 🔬 내부 동작 원리

### 시그널 처리 메커니즘

```
[시그널 발생 경로]

방법 1: kill() 시스템 콜
  kill(target_pid, SIGTERM)
       │
       ▼
  커널: target_pid의 task_struct.pending에 시그널 추가
       → 태스크의 TIF_SIGPENDING 플래그 설정

방법 2: 하드웨어 예외 (예: Segfault → SIGSEGV)
  CPU: 잘못된 메모리 접근 예외 발생
       │
       ▼
  커널 예외 핸들러 → 현재 프로세스에 SIGSEGV 전달

[시그널 처리 시점]
  시그널은 즉시 처리되지 않는다!

  프로세스가 커널 모드 → 유저 모드로 복귀하는 순간 처리:
  (시스템 콜 반환, 인터럽트 핸들러 완료 후)

  TIF_SIGPENDING 확인
         │
         ▼
  시그널 처리 루틴 진입
         │
    ┌────┴────┐
    │         │
  핸들러 없음  핸들러 있음
    │         │
  기본 동작   핸들러 함수 실행
  (종료/무시/중단 등)
```

### SIGTERM vs SIGKILL 차이

```
SIGTERM (15):
  - "정상 종료 요청"
  - 프로세스가 핸들러를 등록할 수 있음
  - 프로세스가 무시할 수도 있음 (signal(SIGTERM, SIG_IGN))
  - 유저 모드 → 커널 모드 → 유저 모드 전환 시 처리
  - 처리 시점: 비결정적 (스케줄링에 따라 수 ms ~ 수백 ms)

SIGKILL (9):
  - "강제 종료 명령"
  - 핸들러 등록 불가능 (커널이 직접 처리)
  - 무시 불가능
  - D 상태(Uninterruptible Sleep)에서는 처리 불가
    (I/O 완료될 때까지 대기)
  - 프로세스 자원 즉시 회수
```

### Kubernetes 종료 흐름

```
kubectl delete pod 실행
       │
       ▼
1. Endpoint에서 Pod IP 제거 (트래픽 차단 시작)
       │
       ▼
2. preStop 훅 실행 (설정된 경우)
   예: sleep 5 (로드 밸런서가 업데이트될 시간 확보)
       │
       ▼
3. SIGTERM을 PID 1에 전달
       │
       ├── Spring Boot: server.shutdown=graceful
       │   → 새 요청 거부, 진행 중 요청 완료 대기
       │   → 완료 후 JVM 종료
       │
       │   (terminationGracePeriodSeconds 타이머 시작)
       │
       ▼
4. terminationGracePeriodSeconds (기본 30초) 후:
   → 아직 살아있으면 SIGKILL 전달
   → 강제 종료

PID 1 문제:
  Unix 관례상 PID 1은 기본 SIGTERM 핸들러가 없음
  → shell script가 PID 1이면 java 자식에게 SIGTERM 전달 안 됨!
  → tini, dumb-init 같은 초소형 init 프로세스로 해결
```

### IPC 메커니즘 비교

```
1. Pipe (익명 파이프)
   ┌──────────────────────────────────────────────────────────┐
   │  [부모 프로세스] ──write──→ [커널 버퍼] ──read──→ [자식 프로세스] │
   └──────────────────────────────────────────────────────────┘
   - 단방향, 관련 프로세스(부모-자식)만 사용 가능
   - 커널 버퍼 크기: 64KB (pipe_buf)
   - 버퍼 가득 차면 write()가 블로킹

2. FIFO (Named Pipe)
   - 파일 시스템에 이름 있는 파이프 (/tmp/my_pipe)
   - 관계없는 프로세스 간 통신 가능
   - 특성은 Pipe와 동일 (단방향, 64KB 버퍼)

3. Message Queue
   - 커널이 관리하는 메시지 큐
   - 메시지 단위로 읽기/쓰기
   - 우선순위 지원
   - 비동기 (읽는 쪽이 없어도 메시지 저장)

4. Shared Memory (공유 메모리)
   ┌───────────────────────────────────────────┐
   │ [프로세스 A]  [공유 메모리 세그먼트]  [프로세스 B] │
   │  가상주소 ──→ [물리 메모리] ←── 가상주소         │
   └───────────────────────────────────────────┘
   - 가장 빠름 (복사 없이 직접 접근)
   - 동기화는 별도 세마포어 필요
   - 대용량 데이터 교환에 적합
   - /dev/shm: tmpfs 기반 공유 메모리 영역

5. Unix Domain Socket
   - 같은 호스트의 프로세스 간 소켓 통신
   - 양방향, 스트림/데이터그램 모두 지원
   - TCP 소켓보다 빠름 (루프백 없음)
   - Docker 컨테이너-호스트 통신에 사용
     (docker.sock: /var/run/docker.sock)
```

---

## 💻 실전 실험

### 실험 1: 시그널 핸들러 등록 및 동작 확인

```c
/* signal_test.c */
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void sigterm_handler(int signo) {
    printf("\nSIGTERM 수신! 정상 종료 중...\n");
    printf("리소스 정리 완료\n");
    _exit(0);   // exit() 대신 _exit() (재진입 안전)
}

int main() {
    signal(SIGTERM, sigterm_handler);
    printf("PID: %d\n", getpid());
    printf("SIGTERM 핸들러 등록 완료. 대기 중...\n");

    while(1) {
        printf("실행 중...\n");
        sleep(1);
    }
    return 0;
}
```

```bash
$ gcc signal_test.c -o signal_test
$ ./signal_test &
PID: 12345

# 별도 터미널에서:
$ kill -TERM 12345
# 출력:
# SIGTERM 수신! 정상 종료 중...
# 리소스 정리 완료

$ kill -9 12345
# 즉시 종료 (핸들러 없음)
```

### 실험 2: 프로세스가 등록한 시그널 확인

```bash
$ PID=$(pgrep -f "java")

$ cat /proc/$PID/status | grep -E "Sig(Blk|Ign|Cgt)"
SigBlk: 0000000000000000   ← 블로킹된 시그널 (비트마스크)
SigIgn: 0000000000000004   ← 무시하는 시그널 (SIGQUIT=3 → 비트2=0x4)
SigCgt: 0000000180014203   ← 캐치하는 시그널

# SIGTERM(15) = 비트 14 = 0x4000
# SigCgt에 0x4000 비트가 설정되어 있으면 SIGTERM 핸들러 있음
$ python3 -c "print(bin(0x0000000180014203))"
# 비트 14(SIGTERM) 확인
```

### 실험 3: Pipe IPC 실험

```bash
# 파이프 버퍼 크기 확인
$ cat /proc/sys/fs/pipe-max-size
1048576  # 최대 1MB (기본 버퍼는 64KB)

# 파이프로 두 프로세스 연결
$ mkfifo /tmp/test_fifo

# 터미널 1: 읽기 대기
$ cat /tmp/test_fifo

# 터미널 2: 데이터 쓰기
$ echo "Hello from producer" > /tmp/test_fifo

# 파이프 버퍼 포화 테스트
$ dd if=/dev/zero bs=1 count=65536 > /tmp/test_fifo &
$ sleep 0.1
$ ls -l /proc/$!/fd/   # 파이프 fd 확인
```

### 실험 4: Docker Graceful Shutdown 확인

```bash
# Dockerfile 예시
cat > Dockerfile << 'EOF'
FROM openjdk:21-slim
COPY app.jar /app.jar
# shell 형식 (PID 1 = sh → SIGTERM이 java에 전달 안 됨!)
CMD java -jar /app.jar

# exec 형식 (PID 1 = java → SIGTERM 직접 수신)
# ENTRYPOINT ["java", "-jar", "/app.jar"]
EOF

$ docker run -d --name test-app myapp
$ docker exec test-app ps aux
# shell 형식 시:
# PID 1 = /bin/sh -c java -jar ...
# PID 7 = java -jar ...
# → docker stop이 sh(PID 1)에 SIGTERM → java에 전달 안 됨

# exec 형식 시:
# PID 1 = java -jar ...
# → docker stop이 java에 직접 SIGTERM

$ time docker stop test-app
# shell 형식: 10초 후 SIGKILL (Graceful Shutdown 안 됨)
# exec 형식: 즉시 종료 (Graceful Shutdown 성공)
```

---

## 📊 성능/비용 비교

| IPC 방법 | 속도 | 용량 제한 | 양방향 | 관계없는 프로세스 | 사용 사례 |
|---------|-----|---------|-------|---------------|---------|
| Pipe | 빠름 | 64KB 버퍼 | X | X | shell 파이프라인 |
| FIFO | 빠름 | 64KB 버퍼 | X | O | 단방향 데이터 스트림 |
| Message Queue | 중간 | 설정 가능 | O | O | 비동기 메시지 전달 |
| Shared Memory | 최빠름 | OS 메모리 한도 | O | O | 대용량 데이터 공유 |
| Unix Domain Socket | 빠름 | 설정 가능 | O | O | 범용 IPC, Docker |
| TCP 소켓 | 느림 | 설정 가능 | O | O (네트워크) | 분산 시스템 |

---

## ⚖️ 트레이드오프

**SIGTERM + Graceful Shutdown의 트레이드오프**:

Graceful Shutdown은 데이터 일관성과 사용자 경험을 위해 필수적이지만, 종료 시간이 길어지는 단점이 있다. `terminationGracePeriodSeconds`를 너무 길게 설정하면 배포 시간이 늘어나고, 너무 짧게 설정하면 진행 중인 요청이 끊긴다. 서비스의 P99 응답 시간을 기준으로 적절한 값을 설정해야 한다.

**공유 메모리의 장단점**:

공유 메모리는 복사 없이 두 프로세스가 직접 메모리를 공유해 가장 빠른 IPC다. 하지만 동기화(세마포어, 뮤텍스)를 직접 구현해야 하고, 한 프로세스가 잘못된 쓰기를 하면 다른 프로세스의 데이터가 손상된다. Redis 같은 서버가 공유 메모리보다 소켓 기반 IPC를 선택하는 이유다.

---

## 📌 핵심 정리

```
시그널:
  SIGTERM (15): 정상 종료 요청, 핸들러 등록/무시 가능
  SIGKILL  (9): 강제 종료, 핸들러 불가, D 상태에서도 안 먹힘
  처리 시점: 커널 → 유저 모드 복귀 순간

Graceful Shutdown 체인:
  Kubernetes → preStop 훅 → SIGTERM → 애플리케이션 → 정리 → 종료
  → terminationGracePeriodSeconds 내 완료 안 되면 SIGKILL

Docker PID 1 문제:
  CMD ["sh", "-c", "java -jar app.jar"]  → sh가 PID 1 (SIGTERM 전달 안 됨)
  ENTRYPOINT ["java", "-jar", "app.jar"] → java가 PID 1 (SIGTERM 직접 수신)

IPC 선택 기준:
  단순 단방향 스트림 → Pipe / FIFO
  비동기 메시지    → Message Queue
  대용량 데이터    → Shared Memory (동기화 필요)
  범용 양방향      → Unix Domain Socket

진단 명령어:
  /proc/<pid>/status의 SigCgt → SIGTERM 핸들러 등록 여부 확인
  kill -0 <pid>              → 프로세스 존재 확인
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Boot 앱이 컨테이너에서 `SIGTERM`을 받았다. `server.shutdown=graceful` 설정이 있고 현재 처리 중인 요청이 있다. `terminationGracePeriodSeconds=10`인데 요청 처리에 15초가 걸린다면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

10초 후 Kubernetes가 `SIGKILL`을 보내 강제 종료된다. 처리 중이던 요청은 중간에 끊긴다. 해결책은 `terminationGracePeriodSeconds`를 최대 예상 요청 처리 시간보다 충분히 길게 설정하거나, `spring.lifecycle.timeout-per-shutdown-phase`를 늘리는 것이다. 또한 `preStop` 훅에 `sleep 5`를 추가해 로드 밸런서가 해당 Pod를 제거할 시간을 확보하면 새 요청이 들어오지 않아 더 빨리 Graceful Shutdown이 완료된다.

</details>

**Q2.** 프로세스가 `SIGTERM`을 무시(SIG_IGN)하도록 설정했다. 이 프로세스를 종료하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

`SIGKILL`을 사용해야 한다. `kill -9 <pid>`로 전달한다. `SIGKILL`은 어떤 프로세스도 무시하거나 핸들러를 등록할 수 없다. 단, 프로세스가 `D` 상태(Uninterruptible Sleep)라면 `SIGKILL`도 즉시 처리되지 않고 I/O가 완료될 때까지 대기한다.

</details>

**Q3.** `docker.sock`(Unix Domain Socket)를 통해 Docker CLI가 Docker daemon과 통신한다. 이것이 TCP 소켓보다 빠른 이유는?

<details>
<summary>해설 보기</summary>

Unix Domain Socket은 같은 호스트 내의 통신이므로 네트워크 스택을 거치지 않는다. TCP의 경우 패킷을 IP 헤더로 감싸고, 체크섬 계산, 루프백 인터페이스 처리 등을 거쳐야 한다. Unix Domain Socket은 이 과정 없이 커널 내부 버퍼에서 직접 데이터를 전달한다. 또한 TCP는 연결 수립을 위해 3-way handshake가 필요하지만, Unix Domain Socket은 파일처럼 `open()`으로 즉시 연결할 수 있다.

</details>

---

**[⬅️ 이전: 컨텍스트 스위칭 비용](./04-context-switching.md)** | **[홈으로 🏠](../README.md)** | **[다음: CFS 스케줄러 ➡️](./06-cfs-scheduler.md)**
