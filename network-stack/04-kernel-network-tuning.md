# 04. 커널 네트워크 파라미터 튜닝 — 실전 설정

## 🎯 핵심 질문

- `net.core.somaxconn`, `tcp_max_syn_backlog`, `tcp_tw_reuse`는 각각 어떤 커널 동작을 바꾸는가?
- `TIME_WAIT` 소켓이 수만 개 쌓이는 원인과 진단 방법은?
- `ulimit -n` 파일 디스크립터 한도가 네트워크 성능과 어떻게 연결되는가?
- 고트래픽 서버의 커널 네트워크 파라미터 최적 설정 체크리스트는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**API 서버에서 `Too many open files` 오류가 발생하며 새 연결을 받지 못한다.** 소켓도 fd이므로 `ulimit -n`(기본 1024)이 소켓 수를 제한한다. 동시 연결 1000개 + 로그 파일 + DB 연결 + 기타를 합산하면 1024는 금방 차버린다.

**배포 후 새 서버가 이전 IP:PORT 조합을 재사용 못 해서 `bind() failed: Address already in use` 오류가 난다.** `SO_REUSEADDR`와 `tcp_tw_reuse`의 차이를 모르면 왜 TIME_WAIT 소켓이 `bind()`를 막는지 이해할 수 없다.

**마이크로서비스 환경에서 서비스 A → B → C 체인 호출 시 TIME_WAIT가 폭발한다.** 각 호출이 새 TCP 연결을 만들고 끊으면 4-tuple이 TIME_WAIT로 고갈된다. HTTP Keep-Alive가 핵심 해결책이고, `tcp_tw_reuse`는 보조 수단이다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: ulimit -n 설정 안 하고 기본값 1024
$ ulimit -n
1024   ← Nginx가 1024개 연결 + 정적 파일 fd + 로그 fd
# 연결이 900개만 돼도 "Too many open files"

# 실수 2: tcp_tw_reuse를 만병통치약으로 사용
$ sysctl -w net.ipv4.tcp_tw_reuse=1
# TIME_WAIT 소켓을 재사용 가능하게 함 (클라이언트 측)
# 하지만 서버 측 TIME_WAIT는 주소를 재사용하는 게 아님
# → 근본 원인(짧은 연결 반복) 해결 안 됨

# 실수 3: tcp_tw_recycle 활성화 (절대 금지)
$ sysctl -w net.ipv4.tcp_tw_recycle=1   # 리눅스 4.12에서 제거됨
# NAT 환경에서 연결 오류 유발
# 여러 클라이언트가 같은 NAT IP 사용 시 timestamp 검사로 드롭
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 고트래픽 서버 기본 설정 체크리스트

# 1. 파일 디스크립터 한도
$ cat /proc/sys/fs/file-max          # 시스템 전체 한도
$ ulimit -n                           # 프로세스별 소프트 한도
# 영구 설정:
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf

# 2. 소켓 큐 크기
$ sysctl net.core.somaxconn           # Accept Queue 상한 (기본 128)
$ sysctl net.ipv4.tcp_max_syn_backlog # SYN Backlog (기본 1024)
# 설정:
sysctl -w net.core.somaxconn=4096
sysctl -w net.ipv4.tcp_max_syn_backlog=8192

# 3. TIME_WAIT 관리
$ sysctl net.ipv4.tcp_tw_reuse       # TIME_WAIT 재사용 (클라이언트)
$ sysctl net.ipv4.tcp_fin_timeout    # FIN_WAIT_2 타임아웃 (기본 60초)
# 설정:
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=30

# 4. 소켓 버퍼 크기
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem="4096 131072 134217728"
sysctl -w net.ipv4.tcp_wmem="4096 131072 134217728"

# 영구 적용
cat >> /etc/sysctl.conf << 'EOF'
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 131072 134217728
net.ipv4.tcp_wmem = 4096 131072 134217728
EOF
$ sysctl -p  # 즉시 적용
```

---

## 🔬 내부 동작 원리

### 주요 파라미터와 커널 동작

```
1. net.core.somaxconn
   → listen() backlog의 최대 상한
   → 실제 Accept Queue 크기 = min(somaxconn, listen() backlog)
   기본값: 128 (매우 작음!)
   권장: 4096 이상 (트래픽에 따라 조정)

2. net.ipv4.tcp_max_syn_backlog
   → SYN Backlog 최대 크기
   → 3-way handshake 진행 중인 half-open 연결 수
   기본값: 1024 (보통 충분)
   권장: 8192 (급증 트래픽 대비)

3. net.ipv4.tcp_tw_reuse
   → TIME_WAIT 상태 소켓의 4-tuple 재사용 허용
   → 클라이언트 측 새 연결 시 적용 (connect() 호출)
   → 타임스탬프 확인으로 안전 보장 (tcp_timestamps 필요)
   기본값: 0 (비활성)
   권장: 1 (단기 연결이 많은 API 서버)

   주의: tcp_tw_reuse vs tcp_tw_recycle
   tcp_tw_reuse=1:   타임스탬프 기반 안전한 재사용 ← 사용 OK
   tcp_tw_recycle=1: 빠른 TIME_WAIT 제거 (NAT 환경 위험)
                     리눅스 4.12에서 완전히 제거됨

4. net.ipv4.tcp_fin_timeout
   → FIN_WAIT_2 상태 최대 유지 시간
   → 상대방이 FIN을 보내지 않는 경우 대기 시간
   기본값: 60초
   권장: 30초 (빠른 리소스 해제)

5. net.ipv4.ip_local_port_range
   → 클라이언트 소켓의 source port 범위
   기본값: 32768~60999 (약 28231개)
   → 동시 TIME_WAIT가 이 수를 초과하면 포트 고갈
   권장: 1024~65535 (약 64511개)
   sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

### TIME_WAIT 깊이 이해

```
TIME_WAIT가 필요한 이유:

시나리오 없이:
  t=0: Connection A (src:50001 → dst:8080) 종료
  t=1: 지연된 패킷이 아직 네트워크에 있음
  t=2: 새 Connection B (src:50001 → dst:8080) 생성
  t=3: 지연된 패킷이 Connection B에 도착!
  → 데이터 오염

TIME_WAIT (2 * MSL = 60초):
  지연 패킷의 최대 수명(MSL) * 2 동안 같은 4-tuple 재사용 금지
  → 오래된 패킷이 새 연결에 영향을 줄 수 없음

TIME_WAIT 대량 발생 패턴:
  마이크로서비스 체인: A → B → C → D (각각 새 TCP 연결)
  초당 100 req, 각 연결 1개
  → 초당 100개 TIME_WAIT 생성
  → 60초 동안 유지
  → 최대 6000개 TIME_WAIT 동시 존재

  포트 범위 28231개 / 6000개 = 포트 고갈까지 여유
  → 초당 400 req라면: 24000개 TIME_WAIT → 포트 범위 초과!
  → "Cannot assign requested address" 오류

진단:
$ ss -s
TCP:   10500 (estab 500, closed 5000, orphaned 100, timewait 5000)

$ ss -tan state time-wait | wc -l
5000

$ ss -tan state time-wait | awk '{print $5}' | \
  cut -d: -f1 | sort | uniq -c | sort -rn | head
# 어느 목적지로의 TIME_WAIT가 많은지 확인
```

### fd 한도와 소켓의 관계

```
소켓 fd 사용 경로:

프로세스 시작 시:
  fd 0: stdin
  fd 1: stdout
  fd 2: stderr
  fd 3+: 열리는 파일/소켓/파이프

소켓 하나당 fd 1개 소비:
  서버 소켓 listen():  fd 1개
  클라이언트 연결 accept(): fd 1개씩

Nginx 1000개 연결 시 fd 사용 예:
  listen 소켓: 2개 (80, 443)
  클라이언트 연결: 1000개
  upstream 연결: 1000개 (backend pool)
  로그 파일: 2개 (access, error)
  설정 파일: 1개
  합계: ~2005개

ulimit -n = 1024이면:
  2005 > 1024 → "Too many open files"

시스템 레벨 한도 (file-max):
  전체 프로세스 합산 fd 수 제한
  일반적으로 매우 크므로 문제 안 됨
  프로세스별 ulimit이 실질적 제한
```

### 포트 범위와 연결 고갈

```
TCP 연결 식별자 (4-tuple):
  (src_ip, src_port, dst_ip, dst_port)

클라이언트가 같은 서버에 연결 시 고갈 가능한 자원:
  src_port 범위: ip_local_port_range
  기본: 32768~60999 = 28231개

마이크로서비스 A → B 호출 예:
  ip_local_port_range = 28231
  TIME_WAIT 유지 = 60초
  포트 고갈까지 허용 RPS = 28231 / 60 ≈ 470 req/s

100 req/s일 때: TIME_WAIT 100 * 60 = 6000개, 안전
470 req/s 이상: 포트 고갈 위험!

해결:
  1. ip_local_port_range 확장 (1024~65535 = 64511개)
  2. tcp_tw_reuse=1 (타임스탬프 기반 재사용)
  3. HTTP Keep-Alive (연결 재사용 → 새 연결 불필요)
  4. Connection Pool (기존 연결 재사용)
```

---

## 💻 실전 실험

### 실험 1: TIME_WAIT 실시간 모니터링

```bash
# TIME_WAIT 추적 스크립트
$ cat << 'EOF' > /tmp/tw_monitor.sh
#!/bin/bash
echo "=== TIME_WAIT 모니터링 ==="
while true; do
    tw=$(ss -tan state time-wait | wc -l)
    estab=$(ss -tan state established | wc -l)
    ports_used=$(ss -tan | awk 'NR>1{split($4,a,":"); print a[length(a)]}' | sort -u | wc -l)
    echo "$(date '+%H:%M:%S') TIME_WAIT=$tw ESTAB=$estab PORTS_USED=$ports_used"
    sleep 1
done
EOF
$ bash /tmp/tw_monitor.sh &

# 단기 연결을 많이 만드는 부하 테스트
$ ab -n 10000 -c 100 http://localhost:8080/api/ping

# TIME_WAIT가 급증하는 것을 관찰
```

### 실험 2: fd 한도 초과 재현

```bash
# fd 한도를 작게 설정하고 재현
$ ulimit -n 50  # 프로세스 fd 50개로 제한

# 소켓을 많이 여는 프로그램
$ python3 -c "
import socket
sockets = []
for i in range(60):
    try:
        s = socket.socket()
        sockets.append(s)
        print(f'소켓 {i+1}개 생성 성공')
    except OSError as e:
        print(f'소켓 {i+1}: 실패 - {e}')
        break
" 
# [Errno 24] Too many open files 발생 시점 확인

# 원래대로 복구
$ ulimit -n 65536

# 현재 프로세스의 fd 사용량 확인
$ ls /proc/$$/fd | wc -l
$ cat /proc/sys/fs/file-nr  # 시스템 전체: 사용중 / 최대
```

### 실험 3: sysctl 설정 전후 비교

```bash
# 기본 설정으로 부하 테스트
$ sysctl net.core.somaxconn
net.core.somaxconn = 128

# Accept Queue 포화 시뮬레이션 (backlog=5로 작게)
$ python3 -c "
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('', 9997))
s.listen(5)
print('listen 시작, accept 5초 지연')
time.sleep(5)
while True: s.accept()[0].close()
" &

$ ab -n 100 -c 50 http://localhost:9997/ 2>&1 | grep "Failed requests"
# Failed requests: 45 (Accept Queue 포화로 실패)

# somaxconn 늘리기
$ sysctl -w net.core.somaxconn=4096
$ kill %1

# 동일 테스트 재실행
# Failed requests: 0 (개선됨)
```

---

## 📊 성능/비용 비교

**고트래픽 서버 파라미터 설정 가이드**:

```
파라미터                    기본값    권장값       효과
net.core.somaxconn         128      4096       Accept Queue 확장
tcp_max_syn_backlog         1024     8192       SYN Backlog 확장
tcp_tw_reuse                0        1          TIME_WAIT 재사용
tcp_fin_timeout            60       30          FIN_WAIT_2 단축
ip_local_port_range     32768-60999 1024-65535  클라이언트 포트 확장
ulimit -n                  1024     65536       fd 한도 확장
tcp_rmem (max)           6291456   134217728    수신 버퍼 확장
tcp_wmem (max)           4194304   134217728    송신 버퍼 확장
```

**TIME_WAIT 수와 성능 관계**:

```
TIME_WAIT 1000개 이하:  영향 없음
TIME_WAIT 5000개:       포트 범위의 18% 소비, 주의 모니터링
TIME_WAIT 20000개:      포트 범위의 71% 소비, 위험 수준
TIME_WAIT 28000개+:     포트 고갈, "Cannot assign requested address"

tcp_tw_reuse=1 적용 시:
  TIME_WAIT 소켓의 포트를 새 연결에서 재사용 가능
  → 효과적 포트 범위: 무제한에 가까움
  → 단, tcp_timestamps=1 (기본)이어야 안전
```

---

## ⚖️ 트레이드오프

**somaxconn 증가의 의미**

Accept Queue를 크게 늘리면 트래픽 급증 시 연결을 더 많이 수용하지만, 서버가 실제로 처리 못 하는 상황에서 연결을 계속 받아들여 클라이언트가 오래 기다리게 된다. 서킷 브레이커 패턴과 함께 사용해 과부하 시 빠른 실패(fail-fast)를 유도하는 것이 더 나은 아키텍처다.

**tcp_tw_reuse와 안전성**

`tcp_tw_reuse=1`은 RFC 1323 타임스탬프를 기반으로 같은 4-tuple의 이전 패킷과 새 패킷을 구별한다. 타임스탬프가 비활성화됐거나 clock이 불일치하면 예상치 못한 패킷이 새 연결에 영향을 줄 수 있다. `tcp_timestamps=1`(기본)인 환경에서는 안전하다.

---

## 📌 핵심 정리

```
핵심 파라미터 요약:
  somaxconn      → Accept Queue 최대 크기 (기본 128, 권장 4096)
  tcp_max_syn_backlog → SYN Backlog (기본 1024, 권장 8192)
  tcp_tw_reuse   → TIME_WAIT 재사용 (클라이언트 측, 권장 1)
  tcp_fin_timeout → FIN_WAIT_2 타임아웃 (기본 60, 권장 30)
  ip_local_port_range → 클라이언트 포트 범위 (권장 1024~65535)
  ulimit -n      → 프로세스 fd 한도 (기본 1024, 권장 65536)

TIME_WAIT 급증 진단:
  ss -s → timewait 수 확인
  ss -tan state time-wait → 상세 목록
  ss -tan | awk '{print $5}' → 목적지 IP 분포

근본 해결책 (파라미터 튜닝보다 우선):
  HTTP Keep-Alive 활성화 (연결 재사용)
  Connection Pool 사용
  단기 연결 줄이기

설정 영구 적용:
  /etc/sysctl.conf 파일에 추가 → sysctl -p
  ulimit: /etc/security/limits.conf
  서비스: systemd unit의 LimitNOFILE
```

---

## 🤔 생각해볼 문제

**Q1.** Kubernetes Pod에서 `ulimit -n`을 늘리려면 어떻게 해야 하는가? OS의 `/etc/security/limits.conf` 수정으로 충분한가?

<details>
<summary>해설 보기</summary>

충분하지 않다. Kubernetes Pod는 컨테이너 환경에서 실행되며, `/etc/security/limits.conf`는 PAM 설정으로 로그인 세션에 적용된다. 컨테이너에는 적용되지 않는다. Pod의 `ulimit`을 높이려면 Kubernetes `securityContext`의 `rlimits` 설정을 사용하거나, 컨테이너 이미지의 `Dockerfile`에서 `ulimit`을 설정해야 한다. systemd로 서비스를 실행한다면 `[Service]` 섹션의 `LimitNOFILE` 옵션을 사용한다.

</details>

**Q2.** `tcp_tw_reuse=1`을 설정했는데 서버 측에서도 TIME_WAIT를 빨리 제거하고 싶다. `tcp_tw_reuse`가 서버 측에도 적용되는가?

<details>
<summary>해설 보기</summary>

`tcp_tw_reuse`는 **클라이언트 측에만** 적용된다. 즉, `connect()` 시스템 콜로 새 연결을 만드는 쪽에서만 효과가 있다. 서버 측 TIME_WAIT는 `tcp_tw_reuse`로 제거할 수 없다. 서버 측에서 TIME_WAIT를 빠르게 줄이려면 `tcp_fin_timeout`을 낮추거나, HTTP Keep-Alive로 연결을 재사용해 TIME_WAIT 자체가 덜 생기도록 해야 한다.

</details>

**Q3.** 마이크로서비스 A가 B를 초당 1000번 호출한다. 각 호출이 새 TCP 연결을 생성한다면 몇 분 안에 포트가 고갈되는가?

<details>
<summary>해설 보기</summary>

기본 `ip_local_port_range=32768~60999`로 28231개 포트 사용 가능하다. 초당 1000개 TIME_WAIT 생성, 60초 유지 → 최대 60,000개 TIME_WAIT 동시 존재가 필요하다. 28231 < 60,000이므로 **약 28초 만에 포트 고갈**이 발생한다. 이때부터 새 연결 시 `Cannot assign requested address` 오류가 발생한다. 해결책은 HTTP Keep-Alive나 Connection Pool로 연결을 재사용하는 것이다. `tcp_tw_reuse=1`과 `ip_local_port_range=1024~65535`로 완화할 수 있지만 근본 해결책은 연결 재사용이다.

</details>

---

**[⬅️ 이전: 소켓 옵션](./03-socket-options.md)** | **[홈으로 🏠](../README.md)** | **[다음: Zero-copy sendfile() ➡️](./05-zero-copy-sendfile.md)**
