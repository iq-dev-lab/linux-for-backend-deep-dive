# 03. 컨테이너 네트워킹 — veth pair와 iptables

## 🎯 핵심 질문

- Docker `bridge` 네트워크가 `veth pair`로 컨테이너와 호스트를 연결하는 방식은?
- `-p 8080:80` 포트 매핑이 커널 iptables 규칙으로 어떻게 구현되는가?
- 두 컨테이너 사이의 통신 경로는 어떻게 되는가?
- 컨테이너 네트워킹의 성능 오버헤드는 얼마나 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

**같은 Docker bridge 네트워크의 컨테이너끼리는 통신이 되는데, 다른 bridge 네트워크끼리는 안 된다.** 각 `docker network create`는 별도의 bridge 인터페이스와 iptables 규칙을 만든다. bridge 간 통신은 기본적으로 iptables의 `DOCKER-ISOLATION` 체인이 차단한다.

**Kubernetes에서 CNI(Container Network Interface)가 필요한 이유는?** 기본 Docker bridge 네트워킹은 단일 호스트 내에서만 동작한다. 다른 노드의 Pod와 통신하려면 오버레이 네트워크(Flannel, Calico, Cilium 등)가 필요하다. 이들은 모두 Network namespace + veth pair + 커널 라우팅을 조합해 구현된다.

**컨테이너에서 외부 인터넷으로 나가는 트래픽이 어떤 경로를 거치는가?** veth pair → docker0 bridge → iptables MASQUERADE(SNAT) → 호스트 eth0 → 외부. NAT 규칙을 이해하면 컨테이너 네트워크 트러블슈팅이 가능하다.

---

## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)

```bash
# 실수 1: 컨테이너 IP로 외부에서 직접 접근 시도
$ docker inspect myapp --format '{{.NetworkSettings.IPAddress}}'
172.17.0.2  ← 컨테이너 내부 IP

$ curl http://172.17.0.2:8080  # 호스트에서는 됨
# 하지만 외부 클라이언트에서:
$ curl http://server-host:172.17.0.2:8080  # 불가능!
# → 컨테이너 IP는 호스트 내부 bridge 네트워크 주소
# → 외부 접근은 반드시 -p로 포트 매핑 필요

# 실수 2: iptables 규칙을 모르고 방화벽만 설정
$ ufw allow 8080  # Ubuntu 방화벽 허용
# Docker는 iptables를 직접 관리하므로 ufw 설정이 우회될 수 있음!
$ iptables -t nat -L DOCKER  # Docker가 추가한 NAT 규칙 확인 필요

# 실수 3: 컨테이너 간 통신에 호스트 IP 사용
# "컨테이너 A에서 컨테이너 B 접근 시 host IP 사용"
$ curl http://192.168.1.100:8080  # 호스트 eth0 IP 사용
# → 비효율적: 컨테이너 A → docker0 → NAT → eth0 → NAT → docker0 → 컨테이너 B
# → 올바른 방법: 같은 네트워크이면 컨테이너 이름으로 접근
$ curl http://container-b:8080  # Docker DNS 사용
```

---

## ✨ 올바른 접근 (After — 커널을 알고 난 후의 설계/운영)

```bash
# 컨테이너 네트워크 전체 구조 파악
$ ip link show | grep -E "docker0|veth"
3: docker0: <BROADCAST,MULTICAST,UP> ... 172.17.0.1/16
7: vethd3c9a1b: <BROADCAST,MULTICAST,UP> master docker0
9: veth2e8f4c0: <BROADCAST,MULTICAST,UP> master docker0

$ brctl show docker0  # bridge 멤버 확인
bridge name  interfaces
docker0      vethd3c9a1b  veth2e8f4c0

# iptables 규칙 확인 (포트 매핑 및 NAT)
$ iptables -t nat -L -n --line-numbers
Chain PREROUTING
1    DOCKER  all  -- 0.0.0.0/0  0.0.0.0/0  ADDRTYPE match dst-type LOCAL

Chain DOCKER
1    DNAT  tcp  -- 0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:80

Chain POSTROUTING
1    MASQUERADE  all  -- 172.17.0.0/16  !172.17.0.0/16

# 특정 컨테이너의 네트워크 경로 추적
$ docker exec mycontainer traceroute 8.8.8.8
1  172.17.0.1  (docker0)  0.123 ms
2  192.168.1.1 (호스트 게이트웨이)  1.234 ms
3  ...
```

---

## 🔬 내부 동작 원리

### veth pair 구조

```
veth pair = 가상 이더넷 인터페이스 쌍
  한쪽에 들어간 패킷이 다른 쪽에서 나옴 (파이프처럼)

컨테이너 생성 시:

호스트 Network namespace:
  ┌──────────────────────────────────────────────┐
  │                                              │
  │  docker0 (bridge)     172.17.0.1/16          │
  │    │                                         │
  │  vethd3c9a1b ─────────────────────────────── │──┐
  │                                              │  │ (veth pair 연결)
  └──────────────────────────────────────────────┘  │
                                                    │
컨테이너 Network namespace:                           │
  ┌──────────────────────────────────────────────┐  │
  │                                              │  │
  │  eth0          172.17.0.2/16  ───────────────│──┘
  │  (컨테이너 내부에서 보이는 인터페이스)                │
  │                                              │
  └──────────────────────────────────────────────┘

패킷 흐름 (컨테이너 → 외부):
  컨테이너 eth0 → veth pair → vethd3c9a1b → docker0 → eth0 → 외부
```

### -p 8080:80 포트 매핑의 iptables 구현

```
docker run -p 8080:80 myapp

커널에 추가되는 iptables 규칙:

1. PREROUTING DNAT (외부 → 컨테이너):
   -t nat -A DOCKER -p tcp --dport 8080 \
     -j DNAT --to-destination 172.17.0.2:80
   
   효과: 호스트의 8080으로 들어온 패킷 → 컨테이너 172.17.0.2:80으로 변환

2. FORWARD (패킷 포워딩 허용):
   -A DOCKER -d 172.17.0.2/32 -p tcp --dport 80 -j ACCEPT
   
   효과: 변환된 패킷이 docker0을 통해 컨테이너로 전달 허용

3. POSTROUTING MASQUERADE (컨테이너 → 외부):
   -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
   
   효과: 컨테이너 IP → 호스트 IP로 SNAT (외부에서 응답 가능)

4. OUTPUT DNAT (호스트 자신에서 접근 시):
   -t nat -A OUTPUT -p tcp --dport 8080 \
     -j DNAT --to-destination 172.17.0.2:80

외부 클라이언트 → 서버:8080 접근 경로:
  [클라이언트] → 서버:8080
    → PREROUTING DNAT: 목적지 → 172.17.0.2:80
    → FORWARD: 허용
    → docker0 → veth → 컨테이너 eth0 → 앱

컨테이너 응답 경로:
  컨테이너 → docker0 → POSTROUTING MASQUERADE(src:컨테이너IP → src:서버IP) → 클라이언트
```

### 컨테이너 간 통신 경로

```
같은 bridge 네트워크 (컨테이너 A → 컨테이너 B):

  컨테이너 A (172.17.0.2)
    │ eth0 → veth pair A
    ▼
  docker0 bridge (L2 스위칭)
    │ veth pair B → eth0
    ▼
  컨테이너 B (172.17.0.3)

  → 호스트 IP 거치지 않음
  → iptables에서 FORWARD 체인 허용 필요

다른 bridge 네트워크 간 통신 (기본: 차단):
  docker network create net1 → 172.18.0.0/16 → br-net1
  docker network create net2 → 172.19.0.0/16 → br-net2

  iptables DOCKER-ISOLATION 체인:
    -A DOCKER-ISOLATION-STAGE-1 -i br-net1 ! -o br-net1 -j DOCKER-ISOLATION-STAGE-2
    -A DOCKER-ISOLATION-STAGE-2 -o br-net2 -j DROP
    → net1 → net2 차단!

  허용하려면: 컨테이너를 두 네트워크에 모두 연결
  $ docker network connect net2 container-in-net1
```

---

## 💻 실전 실험

### 실험 1: veth pair와 bridge 구조 확인

```bash
$ docker run -d --name net-test1 nginx
$ docker run -d --name net-test2 nginx

# 생성된 veth pair 확인
$ ip link show
# vethXXXXXXXX: <...> master docker0

# bridge 멤버 확인
$ bridge link show
vethd3c9a1b master docker0
veth2e8f4c0 master docker0

# 컨테이너 IP 확인
$ docker inspect net-test1 --format '{{.NetworkSettings.IPAddress}}'
172.17.0.2
$ docker inspect net-test2 --format '{{.NetworkSettings.IPAddress}}'
172.17.0.3

# 컨테이너 A에서 컨테이너 B로 ping (bridge를 통해)
$ docker exec net-test1 ping -c 3 172.17.0.3
# 응답 확인

# tcpdump로 bridge 트래픽 확인
$ tcpdump -i docker0 -n 'icmp' &
$ docker exec net-test1 ping -c 1 172.17.0.3
# docker0 인터페이스에서 ICMP 트래픽 캡처됨
```

### 실험 2: iptables 규칙 추적

```bash
# 포트 매핑 컨테이너 실행
$ docker run -d -p 8888:80 --name port-test nginx

# 생성된 NAT 규칙 확인
$ iptables -t nat -L DOCKER -n --line-numbers
num  target     prot  source    destination
1    RETURN     all   127.0.0.0/8  ...
2    DNAT       tcp   0.0.0.0/0   0.0.0.0/0    tcp dpt:8888 to:172.17.0.X:80

# 패킷이 실제로 규칙을 따르는지 추적
$ iptables -t nat -A PREROUTING -p tcp --dport 8888 \
  -j LOG --log-prefix "PORT-MAP: "
$ curl http://localhost:8888
$ dmesg | grep "PORT-MAP"
# 실제 패킷이 규칙을 통과하는 것 확인
$ iptables -t nat -D PREROUTING -p tcp --dport 8888 \
  -j LOG --log-prefix "PORT-MAP: "  # 정리
```

### 실험 3: 컨테이너 네트워크 성능 측정

```bash
# 호스트 직접 통신 vs 컨테이너 bridge 통신 지연 비교
$ apt-get install -y iperf3

# 호스트 직접: iperf3 서버 실행
$ iperf3 -s -p 5201 &

# 컨테이너에서 호스트로 (veth + bridge + iptables 경유)
$ docker run --rm ubuntu bash -c \
  "apt-get install -y iperf3 -q && iperf3 -c 172.17.0.1 -p 5201 -t 5"

# host network 모드 (namespace 공유, 오버헤드 없음)
$ docker run --rm --net=host ubuntu bash -c \
  "apt-get install -y iperf3 -q && iperf3 -c localhost -p 5201 -t 5"

# 예상 결과:
# bridge 모드: ~9.5 Gbps (약간 오버헤드)
# host 모드:   ~40 Gbps (네이티브)
```

---

## 📊 성능/비용 비교

**컨테이너 네트워크 모드별 성능**:

```
모드          처리량        지연(RTT)    격리 수준
host          ~40 Gbps     ~0.05ms     없음 (호스트 공유)
bridge        ~9.5 Gbps    ~0.1ms      컨테이너 격리
overlay       ~7 Gbps      ~0.3ms      멀티 노드 격리
(Flannel 등)

패킷 처리 오버헤드:
  bridge 모드: veth 복사 + bridge 처리 + iptables = ~15~20μs 추가
  host 모드:   직접 처리 = 추가 오버헤드 없음

고성능 네트워크 서비스(Redis, Kafka):
  --net=host 또는 Cilium eBPF 기반 CNI 권장
```

**iptables vs nftables vs eBPF (CNI 관점)**:

```
iptables:   Docker 기본, 규칙 수 증가 시 O(N) 매칭
nftables:   iptables 대체, 더 효율적인 규칙 집합
eBPF (Cilium): 커널 내 JIT 컴파일, O(1) 매칭, 최고 성능
  → 대규모 Kubernetes 클러스터에서 Cilium 권장
```

---

## ⚖️ 트레이드오프

**bridge vs host 네트워크 모드**

`--net=host`는 Network namespace를 사용하지 않아 포트 충돌 위험이 있지만 성능이 최고다. 컨테이너가 호스트의 모든 네트워크 인터페이스를 직접 사용하므로 격리가 없다. Redis, Kafka처럼 네트워크 집약적이고 단일 인스턴스로 실행하는 서비스에서는 `--net=host`가 유용하다. 여러 컨테이너가 같은 포트를 사용해야 하면 bridge 모드가 필수다.

**iptables 규칙의 누적**

컨테이너를 많이 실행하면 iptables 규칙이 선형으로 증가한다. 컨테이너 1000개, 각각 10개 규칙이면 10,000개 규칙이 쌓인다. 모든 패킷이 이 규칙을 순서대로 검사하므로 처리량이 저하된다. 대규모 Kubernetes 클러스터에서 Cilium(eBPF) 같은 CNI로 전환하는 이유가 여기에 있다.

---

## 📌 핵심 정리

```
Docker bridge 네트워킹 구조:
  컨테이너 eth0 ←→ veth pair ←→ docker0 bridge ←→ 호스트 eth0

포트 매핑 (-p 8080:80) 커널 구현:
  iptables DNAT: 호스트:8080 → 컨테이너IP:80
  iptables MASQUERADE: 컨테이너IP → 호스트IP (외부 통신 시)

컨테이너 간 통신:
  같은 bridge: veth → docker0 → veth (호스트 IP 불필요)
  다른 bridge: DOCKER-ISOLATION 체인이 기본 차단

트러블슈팅 명령어:
  ip link show           → veth pair 목록
  brctl show docker0     → bridge 멤버
  iptables -t nat -L -n  → NAT 규칙 (포트 매핑)
  tcpdump -i docker0     → bridge 트래픽 캡처
  docker exec ip route   → 컨테이너 라우팅 테이블

성능 최적화:
  --net=host: 네트워크 집약적 서비스 (격리 포기)
  Cilium (eBPF): 대규모 K8s 클러스터 (iptables 대체)
```

---

## 🤔 생각해볼 문제

**Q1.** 컨테이너 A가 컨테이너 B의 이름으로 접근할 수 있다(`curl http://container-b`). 이름 해석은 어디서 일어나는가?

<details>
<summary>해설 보기</summary>

Docker의 내장 DNS 서버(기본 127.0.0.11)가 담당한다. 같은 Docker 사용자 정의 네트워크에 속한 컨테이너는 Docker daemon이 관리하는 DNS 서버를 통해 이름으로 서로를 찾을 수 있다. 기본 `bridge` 네트워크에서는 이 기능이 없다. 컨테이너 내부의 `/etc/resolv.conf`에 `nameserver 127.0.0.11`이 설정되어 있어 모든 DNS 쿼리가 이 내장 서버로 전달된다. Kubernetes에서는 CoreDNS가 같은 역할을 한다.

</details>

**Q2.** `docker run -p 80:80 nginx`를 실행했는데 `ufw deny 80`을 설정해도 외부에서 접근이 된다. 왜인가?

<details>
<summary>해설 보기</summary>

Docker는 `iptables`를 직접 수정하여 `PREROUTING`과 `FORWARD` 체인에 규칙을 추가한다. `ufw`는 `INPUT` 체인을 관리하지만, Docker의 포트 포워딩은 `PREROUTING`에서 DNAT로 처리되어 `INPUT` 체인에 도달하기 전에 컨테이너로 전달된다. 해결책은 Docker daemon 설정에서 `--iptables=false`로 Docker의 iptables 자동 관리를 비활성화하거나, `DOCKER-USER` 체인에 직접 규칙을 추가하는 것이다(Docker는 `DOCKER-USER` 체인을 먼저 처리하도록 설계됨).

</details>

**Q3.** Kubernetes CNI로 Calico를 사용하면 서로 다른 노드의 Pod끼리 어떻게 통신하는가?

<details>
<summary>해설 보기</summary>

Calico는 BGP(Border Gateway Protocol)를 사용해 각 노드의 Pod 서브넷을 다른 노드에 라우팅 정보로 전달한다. 노드 A의 Pod(10.0.1.2)가 노드 B의 Pod(10.0.2.3)에 접근 시, 노드 A의 라우팅 테이블에 "10.0.2.0/24는 노드 B IP를 통해 가라"는 경로가 있다. VXLAN 오버레이 없이 순수 L3 라우팅으로 동작하므로 오버헤드가 적다. 각 노드에서 `ip route`를 실행하면 다른 노드 Pod 서브넷으로의 경로가 보인다. eBPF 기반 Cilium은 이 과정을 커널 내부에서 처리해 더 효율적이다.

</details>

---

**[⬅️ 이전: cgroups v2](./02-cgroups-v2.md)** | **[홈으로 🏠](../README.md)** | **[다음: OOM in Containers ➡️](./04-oom-in-containers.md)**
