# Part 2. Bastion 및 Load Balancer

> **이 파트의 목적**
> Part 1에서 결정한 네트워크 설계를 실제로 구현하는 인프라 컴포넌트인 Bastion을 구성합니다. Bastion은 폐쇄망 환경에서 OpenShift 설치와 운영의 단일 진입점이자 핵심 의존 서비스 집약점입니다. DNS, Load Balancer, Mirror Registry, NTP, Ignition 제공 등 모든 의존 서비스를 한 노드에서 통합 운영합니다.
>
> 이 파트를 학습하고 나면 Bastion에서 동작하는 각 서비스가 OpenShift 설치/운영 라이프사이클의 어느 시점에 어떻게 사용되는지 이해하고, HAProxy 설정 한 줄 한 줄의 의미를 설명할 수 있어야 합니다.

---

## 2.1 Bastion 개요

### 2.1.1 Bastion의 역할

Bastion은 본 PoC에서 다음 역할을 모두 수행합니다.

| 서비스 | 역할 | 사용 시점 |
|---|---|---|
| DNS (BIND/dnsmasq/CoreDNS) | 클러스터 내부 이름 해석 | 설치 전부터 운영 내내 |
| HAProxy | API/MCS/Ingress LB | 설치 전부터 운영 내내 |
| Mirror Registry (quay 또는 mirror-registry) | 이미지 저장소 | 설치 시점, 운영 중 이미지 pull |
| Web Server (nginx/httpd) | Ignition 파일 제공, RHCOS 이미지 제공 | 설치 시점만 |
| NTP/Chrony | 시간 동기화 기준 또는 중계 | 설치 전부터 운영 내내 |
| oc / openshift-install / oc-mirror | 설치 및 운영 도구 | 설치 시점, Day-2 운영 |
| vBMC | VM 기반 BMC 에뮬레이션 (FAR 테스트용) | FAR 검증 단계 |

### 2.1.2 Bastion 단일 노드 설계의 함의

PoC에서 Bastion은 단일 노드이며, 이는 명시적 단일점(Single Point of Failure)입니다. 다음을 인지하고 시작해야 합니다.

- **Bastion 장애 = 클러스터 신규 설치/노드 추가 불가**: 이미지 pull과 ignition 제공이 멈춤
- **Bastion 장애 = 외부 접근 불가**: console과 API LB가 멈춤
- **Bastion 장애 ≠ 기존 클러스터 즉시 중단**: 이미 동작 중인 워크로드는 계속 실행됨 (Pod가 이미지 pull을 다시 시도하기 전까지)
- **etcd, kubelet은 Bastion에 의존하지 않음**: Bastion이 죽어도 master 간 etcd 통신은 영향 없음

> **PoC vs 프로덕션 차이**
> 프로덕션에서는 다음 분리가 권장됩니다.
> - DNS: 기업 표준 DNS 인프라 사용 또는 이중화된 DNS 서버
> - LB: 물리 L4/L7 장비 (Service Ingress LB는 별도 분리)
> - Mirror Registry: 별도 노드, 가능하면 HA 구성
> - Bastion: 관리/설치 도구 전용으로 축소
>
> 본 PoC는 단순성을 우선해 모든 역할을 한 Bastion에 통합합니다. 프로덕션 전환 시 어느 컴포넌트를 어디로 옮길지 명시한 체크리스트는 Part 6에서 다룹니다.

### 2.1.3 Bastion 사양 권장

| 항목 | PoC 최소 | PoC 권장 |
|---|---|---|
| OS | RHEL 9.x 또는 Rocky 9.x | RHEL 9.4+ |
| CPU | 4 vCPU | 8 vCPU |
| Memory | 16 GiB | 32 GiB |
| Disk (OS) | 100 GiB | 200 GiB |
| Disk (Mirror Registry) | 500 GiB | 1 TiB (OCP 4.20 release + Operators) |
| NIC | 2개 (Infra, Service) | 2개 |

Mirror Registry 디스크 크기는 미러링할 이미지 양에 따라 다릅니다. OCP 4.20 release만 약 100GB, Operator catalog 전체 미러링 시 500GB~1TB 수준입니다.

### 2.1.4 Bastion 네트워크 구성

```
Bastion
├─ NIC1: Infra Network (bond0 또는 단일 NIC)
│  ├─ Primary IP: 192.168.10.10
│  ├─ Secondary IP (VIP): 192.168.10.20 (API/MCS)
│  └─ Secondary IP (VIP): 192.168.10.21 (Default Ingress)
│
└─ NIC2: Service Network
   ├─ Primary IP: 192.168.20.10
   └─ Secondary IP (VIP): 192.168.20.20 (Service Ingress)
```

> **왜 VIP를 Bastion의 primary IP와 분리하는가**
> Bastion의 primary IP(.10.10)는 SSH, DNS, Mirror Registry 등 Bastion 자체 서비스에 사용됩니다. LB VIP를 분리해두면 다음 이점이 있습니다.
> 1. **장애 분석 분리**: API LB 장애와 Bastion SSH 장애를 IP로 구분 가능
> 2. **방화벽 정책 단순화**: VIP별로 허용 포트 정책 분리
> 3. **향후 이전 용이**: VIP만 다른 노드로 옮기면 LB 이전 완료 (DNS 변경 불필요)
> 4. **인증서 SAN 명확**: API VIP의 인증서가 노드 IP와 무관

---

## 2.2 Bastion OS 초기 구성

### 2.2.1 시스템 준비

```bash
# 시간대 설정
timedatectl set-timezone Asia/Seoul

# hostname 설정
hostnamectl set-hostname bastion.ocp1.example.com

# 필수 패키지 설치
dnf install -y \
  bind bind-utils \
  haproxy \
  httpd \
  chrony \
  podman \
  skopeo \
  jq \
  vim \
  wget \
  net-tools \
  bash-completion \
  policycoreutils-python-utils

# SELinux는 enforcing 유지 (권장)
getenforce
# 기대 출력: Enforcing
```

### 2.2.2 네트워크 설정

NetworkManager로 두 NIC를 구성합니다.

```bash
# Infra NIC 구성 (예: ens192)
nmcli con add type ethernet con-name infra ifname ens192 \
  ipv4.method manual \
  ipv4.addresses 192.168.10.10/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns "192.168.10.10" \
  ipv4.dns-search "ocp1.example.com" \
  connection.autoconnect yes

# Service NIC 구성 (예: ens224)
nmcli con add type ethernet con-name service ifname ens224 \
  ipv4.method manual \
  ipv4.addresses 192.168.20.10/24 \
  connection.autoconnect yes

# 활성화
nmcli con up infra
nmcli con up service

# 확인
ip addr
ip route
```

> **Service NIC에 gateway를 설정하지 않는 이유**
> Bastion의 default gateway는 Infra(192.168.10.1) 하나입니다. Service NIC에 별도 gateway를 두면 두 개의 default route가 생겨 라우팅이 비결정적이 됩니다.
> Service망과 통신할 일이 있다면 같은 L2이므로 라우팅 테이블에 자동으로 `192.168.20.0/24 dev ens224` 항목이 생기고, 별도 설정 없이 통신됩니다.

### 2.2.3 VIP 할당

VIP는 NetworkManager로 secondary IP로 추가합니다.

```bash
# API/MCS VIP
nmcli con mod infra +ipv4.addresses 192.168.10.20/24

# Default Ingress VIP
nmcli con mod infra +ipv4.addresses 192.168.10.21/24

# Service Ingress VIP
nmcli con mod service +ipv4.addresses 192.168.20.20/24

# 적용
nmcli con up infra
nmcli con up service

# 확인
ip addr show ens192
ip addr show ens224
```

기대 출력 (Infra NIC):
```
inet 192.168.10.10/24 brd 192.168.10.255 scope global ens192
inet 192.168.10.20/24 brd 192.168.10.255 scope global secondary ens192
inet 192.168.10.21/24 brd 192.168.10.255 scope global secondary ens192
```

### 2.2.4 방화벽 정책

PoC에서는 firewalld를 비활성화하거나 필요한 포트만 명시적으로 허용합니다.

```bash
# 옵션 A: PoC 단순성 우선 (firewalld 비활성화)
systemctl disable --now firewalld

# 옵션 B: 명시적 허용 (권장하지 않으나 보안 요구 시)
firewall-cmd --permanent --add-port=53/tcp
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=22623/tcp
firewall-cmd --permanent --add-port=8443/tcp    # Mirror Registry (예시)
firewall-cmd --permanent --add-port=123/udp     # NTP
firewall-cmd --reload
```

> **PoC에서는 옵션 A 권장**
> 폐쇄망 환경이고 노드 IP가 통제되어 있으므로 Bastion에서 firewalld를 끄는 것이 트러블슈팅 시간을 크게 줄여줍니다. 보안은 외부 방화벽 계층에서 처리합니다.

---

## 2.3 DNS 구성

### 2.3.1 DNS의 역할

OpenShift는 다음 이름 해석에 DNS를 의존합니다.

| 이름 패턴 | 용도 | 해석 결과 |
|---|---|---|
| `api.<cluster>.<domain>` | API 서버 외부 접근 | API VIP |
| `api-int.<cluster>.<domain>` | API 서버 내부 접근 (노드 → API) | API VIP (동일) |
| `*.apps.<cluster>.<domain>` | Default Ingress wildcard | Default Ingress VIP |
| `*.svcapps.<cluster>.<domain>` | Service Ingress wildcard | Service Ingress VIP |
| `<hostname>.<cluster>.<domain>` | 각 노드의 hostname | 노드 IP |
| `<hostname>` PTR | 역방향 조회 | 노드 hostname |

> **`api`와 `api-int`의 차이**
> 두 이름은 같은 API VIP를 가리키지만 용도가 분리되어 있습니다.
> - `api.<cluster>.<domain>`: 외부에서 API 서버에 접근할 때 사용. 외부 CA로 서명된 인증서 사용 가능.
> - `api-int.<cluster>.<domain>`: 클러스터 내부 노드들이 API 서버를 호출할 때 사용. 내부 CA로 서명된 인증서 사용.
>
> OpenShift는 두 이름의 인증서를 별도로 관리하므로, 두 DNS 레코드 모두 반드시 등록되어야 합니다. PoC에서는 두 이름이 같은 IP를 가리켜도 무방합니다.

### 2.3.2 BIND 설정 (named)

```bash
# BIND 설정 파일 위치 확인
ls /etc/named.conf /etc/named/ /var/named/
```

`/etc/named.conf` 수정:

```conf
options {
    listen-on port 53 { 127.0.0.1; 192.168.10.10; };
    listen-on-v6 port 53 { none; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";

    allow-query     { any; };
    allow-recursion { 192.168.10.0/24; 192.168.20.0/24; 127.0.0.1; };

    recursion yes;
    forwarders { 8.8.8.8; };   # 폐쇄망이면 제거 또는 사내 DNS

    dnssec-validation no;
};

zone "ocp1.example.com" IN {
    type master;
    file "ocp1.example.com.zone";
    allow-update { none; };
};

zone "10.168.192.in-addr.arpa" IN {
    type master;
    file "10.168.192.in-addr.arpa.zone";
    allow-update { none; };
};
```

### 2.3.3 정방향 zone 파일

`/var/named/ocp1.example.com.zone`:

```dns
$TTL 86400
@   IN  SOA bastion.ocp1.example.com. admin.ocp1.example.com. (
            2026051501  ; Serial (YYYYMMDDNN)
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400 )     ; Minimum TTL
;
@           IN  NS      bastion.ocp1.example.com.
bastion     IN  A       192.168.10.10

; API 엔드포인트
api         IN  A       192.168.10.20
api-int     IN  A       192.168.10.20

; Ingress 와일드카드 (관리용)
*.apps      IN  A       192.168.10.21

; Ingress 와일드카드 (업무용)
*.svcapps   IN  A       192.168.20.20

; Bootstrap (설치 후 제거 가능)
bootstrap   IN  A       192.168.10.30

; Master 노드
master-0    IN  A       192.168.10.31
master-1    IN  A       192.168.10.32
master-2    IN  A       192.168.10.33

; AP Worker
ap-worker-0 IN  A       192.168.10.41
ap-worker-1 IN  A       192.168.10.42

; DB Worker
db-worker-0 IN  A       192.168.10.51
db-worker-1 IN  A       192.168.10.52

; PodPool1 Worker (Infra IP만 등록)
podpool1-worker-0   IN  A   192.168.10.61
podpool1-worker-1   IN  A   192.168.10.62

; NAS
nas         IN  A       192.168.10.70

; Mirror Registry (Bastion과 동일)
mirror      IN  A       192.168.10.10
```

> **PodPool1의 Service IP를 DNS에 등록하지 않는 이유**
> PodPool1 노드는 Infra IP(192.168.10.61)와 Service IP(192.168.20.61)를 모두 가지지만, DNS hostname은 Infra IP 하나로 등록합니다.
> - 노드 hostname은 OpenShift 입장에서 **InternalIP에 매핑되는 이름** 하나만 있으면 됨
> - Service IP는 LB backend로만 사용되므로 별도 hostname 불필요
> - 두 IP에 별도 hostname을 두면 어느 이름으로 SSH/접근해야 하는지 운영 혼동 발생

### 2.3.4 역방향 zone 파일

`/var/named/10.168.192.in-addr.arpa.zone`:

```dns
$TTL 86400
@   IN  SOA bastion.ocp1.example.com. admin.ocp1.example.com. (
            2026051501
            3600
            1800
            604800
            86400 )
;
@   IN  NS  bastion.ocp1.example.com.

10  IN  PTR bastion.ocp1.example.com.
20  IN  PTR api.ocp1.example.com.
21  IN  PTR apps.ocp1.example.com.
30  IN  PTR bootstrap.ocp1.example.com.
31  IN  PTR master-0.ocp1.example.com.
32  IN  PTR master-1.ocp1.example.com.
33  IN  PTR master-2.ocp1.example.com.
41  IN  PTR ap-worker-0.ocp1.example.com.
42  IN  PTR ap-worker-1.ocp1.example.com.
51  IN  PTR db-worker-0.ocp1.example.com.
52  IN  PTR db-worker-1.ocp1.example.com.
61  IN  PTR podpool1-worker-0.ocp1.example.com.
62  IN  PTR podpool1-worker-1.ocp1.example.com.
70  IN  PTR nas.ocp1.example.com.
```

> **역방향 DNS가 왜 중요한가**
> kubelet의 노드 등록, etcd peer 인증서 검증, OpenShift 일부 컴포넌트의 SAN 검증 시 역방향 DNS를 사용합니다. **역방향 등록이 누락되면 설치는 성공해도 노드 인증서가 정확하지 않게 발급되거나 일부 헬스체크가 실패할 수 있습니다.** 정방향만큼 중요하게 다뤄야 합니다.

### 2.3.5 DNS 서비스 시작 및 검증

```bash
# 설정 문법 검증
named-checkconf /etc/named.conf
named-checkzone ocp1.example.com /var/named/ocp1.example.com.zone
named-checkzone 10.168.192.in-addr.arpa /var/named/10.168.192.in-addr.arpa.zone

# 권한 설정 (BIND는 named user로 실행됨)
chown root:named /var/named/ocp1.example.com.zone /var/named/10.168.192.in-addr.arpa.zone
chmod 640 /var/named/ocp1.example.com.zone /var/named/10.168.192.in-addr.arpa.zone

# 서비스 시작
systemctl enable --now named

# 정방향 조회 검증
dig @192.168.10.10 api.ocp1.example.com +short
# 기대: 192.168.10.20

dig @192.168.10.10 console-openshift-console.apps.ocp1.example.com +short
# 기대: 192.168.10.21

dig @192.168.10.10 frontend.svcapps.ocp1.example.com +short
# 기대: 192.168.20.20

dig @192.168.10.10 master-0.ocp1.example.com +short
# 기대: 192.168.10.31

# 역방향 조회 검증
dig @192.168.10.10 -x 192.168.10.31 +short
# 기대: master-0.ocp1.example.com.

dig @192.168.10.10 -x 192.168.10.20 +short
# 기대: api.ocp1.example.com.
```

### 2.3.6 흔한 실수

| 증상 | 원인 |
|---|---|
| `dig api.ocp1.example.com` 결과 없음 | zone 파일 권한 잘못 (named 사용자가 읽을 수 없음) |
| 새 레코드가 반영되지 않음 | Serial 번호를 증가시키지 않음 |
| 노드 hostname이 `localhost`로 보임 | 역방향 DNS 미등록 |
| `console`은 되는데 `oauth-openshift`는 안 됨 | `*.apps` wildcard 미설정 (개별 등록만 함) |

> **Serial 번호 관리 팁**
> zone 파일을 수정할 때마다 Serial 번호를 증가시켜야 합니다. `YYYYMMDDNN` 형식(2026051501, 2026051502, ...)을 사용하면 직관적이고 일별 변경 추적이 가능합니다.

---

## 2.4 HAProxy 구성 (Load Balancer)

### 2.4.1 LB 구성 전략

본 PoC에서 HAProxy는 단일 프로세스 안에서 두 개의 논리적 LB 역할을 수행합니다.

```
HAProxy (Bastion에서 단일 프로세스로 동작)
│
├─ [Infra LB 영역]
│  ├─ frontend api          → 192.168.10.20:6443
│  ├─ frontend machine_config → 192.168.10.20:22623
│  ├─ frontend default_http  → 192.168.10.21:80
│  └─ frontend default_https → 192.168.10.21:443
│
└─ [Service LB 영역]
   ├─ frontend service_http  → 192.168.20.20:80
   └─ frontend service_https → 192.168.20.20:443
```

이 분리는 설정 파일에서 frontend/backend 이름과 주석으로 표현됩니다. 동일 프로세스이지만 **논리적으로는 두 개의 LB가 동작**합니다. 프로덕션 전환 시 service 관련 frontend/backend를 떼어내 별도 LB로 옮기면 됩니다.

### 2.4.2 전체 HAProxy 설정 파일

`/etc/haproxy/haproxy.cfg`:

```haproxy
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     20000
    user        haproxy
    group       haproxy
    daemon
    stats       socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# Common defaults
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 20000

#---------------------------------------------------------------------
# HAProxy statistics (운영자 모니터링용)
#---------------------------------------------------------------------
listen stats
    bind 192.168.10.10:9000
    mode http
    stats enable
    stats uri /
    stats refresh 10s
    stats realm HAProxy\ Statistics
    stats auth admin:RedHat123!     # PoC 예시 비밀번호, 실환경에서는 강한 값 사용

#=====================================================================
# [Infra LB 영역] - 관리 트래픽
#=====================================================================

#---------------------------------------------------------------------
# Kubernetes API Server (6443)
#---------------------------------------------------------------------
frontend api
    bind 192.168.10.20:6443
    mode tcp
    option tcplog
    default_backend api_backend

backend api_backend
    mode tcp
    balance roundrobin
    option tcp-check
    server bootstrap 192.168.10.30:6443 check check-ssl verify none  # 설치 후 주석 처리
    server master-0  192.168.10.31:6443 check check-ssl verify none
    server master-1  192.168.10.32:6443 check check-ssl verify none
    server master-2  192.168.10.33:6443 check check-ssl verify none

#---------------------------------------------------------------------
# Machine Config Server (22623)
#---------------------------------------------------------------------
frontend machine_config
    bind 192.168.10.20:22623
    mode tcp
    option tcplog
    default_backend machine_config_backend

backend machine_config_backend
    mode tcp
    balance roundrobin
    option tcp-check
    server bootstrap 192.168.10.30:22623 check  # 설치 후 주석 처리
    server master-0  192.168.10.31:22623 check
    server master-1  192.168.10.32:22623 check
    server master-2  192.168.10.33:22623 check

#---------------------------------------------------------------------
# Default Ingress HTTP (80) - 관리 트래픽
#---------------------------------------------------------------------
frontend default_ingress_http
    bind 192.168.10.21:80
    mode tcp
    option tcplog
    default_backend default_ingress_http_backend

backend default_ingress_http_backend
    mode tcp
    balance roundrobin
    option httpchk GET /healthz
    http-check expect status 200
    server ap-worker-0 192.168.10.41:80 check port 1936
    server ap-worker-1 192.168.10.42:80 check port 1936

#---------------------------------------------------------------------
# Default Ingress HTTPS (443) - 관리 트래픽
#---------------------------------------------------------------------
frontend default_ingress_https
    bind 192.168.10.21:443
    mode tcp
    option tcplog
    default_backend default_ingress_https_backend

backend default_ingress_https_backend
    mode tcp
    balance roundrobin
    option httpchk GET /healthz
    http-check expect status 200
    server ap-worker-0 192.168.10.41:443 check port 1936
    server ap-worker-1 192.168.10.42:443 check port 1936

#=====================================================================
# [Service LB 영역] - 업무 트래픽
#=====================================================================

#---------------------------------------------------------------------
# Service Ingress HTTP (80) - 업무 트래픽
#---------------------------------------------------------------------
frontend service_ingress_http
    bind 192.168.20.20:80
    mode tcp
    option tcplog
    default_backend service_ingress_http_backend

backend service_ingress_http_backend
    mode tcp
    balance roundrobin
    option httpchk GET /healthz
    http-check expect status 200
    server podpool1-worker-0 192.168.20.61:80 check port 1936
    server podpool1-worker-1 192.168.20.62:80 check port 1936

#---------------------------------------------------------------------
# Service Ingress HTTPS (443) - 업무 트래픽
#---------------------------------------------------------------------
frontend service_ingress_https
    bind 192.168.20.20:443
    mode tcp
    option tcplog
    default_backend service_ingress_https_backend

backend service_ingress_https_backend
    mode tcp
    balance roundrobin
    option httpchk GET /healthz
    http-check expect status 200
    server podpool1-worker-0 192.168.20.61:443 check port 1936
    server podpool1-worker-1 192.168.20.62:443 check port 1936
```

### 2.4.3 설정 요소 해설

**1. `mode tcp` vs `mode http`**

모든 frontend가 `mode tcp`인 이유는 OpenShift 트래픽의 특성 때문입니다.

- **API/MCS**: TLS 종단을 LB에서 풀지 않음 (master까지 그대로 전달). 클라이언트 인증서 통과 필요.
- **Ingress**: TLS는 IngressController에서 처리. LB는 단순 TCP passthrough.

`mode http`로 두면 LB가 패킷을 HTTP로 해석하려 하지만, TLS가 풀려 있지 않으므로 실패합니다.

**2. `option httpchk GET /healthz` + `check port 1936`**

IngressController는 1936 포트에서 stats와 헬스체크 endpoint를 제공합니다. 단순 TCP check(`port 443`)는 노드 OS가 살아있으면 통과하므로, **router Pod가 죽었는데도 backend가 살아있는 것으로 잘못 판단**합니다. 1936/healthz를 사용하면 router Pod의 실제 상태를 정확히 확인합니다.

방화벽 정책 필요: Bastion → ap-worker-0/1:1936, Bastion → podpool1-worker-0/1:1936 허용.

**3. `check-ssl verify none` (API backend)**

API 서버는 6443에서 TLS로 통신합니다. HAProxy가 헬스체크를 TLS로 수행하되 인증서 검증은 생략(self-signed CA이므로).

**4. bootstrap 항목**

bootstrap 노드는 설치 단계에서만 사용됩니다. 설치 완료 후 (대략 30~60분) bootstrap을 LB에서 제거해야 합니다.

```bash
# 설치 완료 후
sed -i 's/^    server bootstrap/#    server bootstrap/' /etc/haproxy/haproxy.cfg
systemctl reload haproxy
```

**5. listen stats (9000)**

운영자가 HAProxy 상태를 웹 UI로 확인할 수 있도록 9000 포트에 통계 페이지 제공. 폐쇄망이라도 운영자 IP만 허용하는 방식으로 사용 권장.

### 2.4.4 HAProxy 시작 및 검증

```bash
# 설정 문법 검증
haproxy -c -f /etc/haproxy/haproxy.cfg

# 서비스 시작
systemctl enable --now haproxy

# 상태 확인
systemctl status haproxy
ss -tlnp | grep haproxy

# 통계 페이지 확인 (브라우저)
# http://192.168.10.10:9000
# admin / RedHat123!
```

> **bootstrap 단계에서는 backend가 모두 DOWN 상태가 정상**
> 설치 시작 전에는 master 노드가 아직 생성되지 않았으므로 HAProxy 통계 페이지에서 모든 master backend가 DOWN으로 표시됩니다. bootstrap이 부팅되어 6443/22623 listen을 시작하면 bootstrap backend부터 UP으로 바뀝니다. 이는 정상 흐름입니다.

### 2.4.5 SELinux 정책

SELinux가 활성화된 상태에서 HAProxy가 비표준 포트(6443, 22623)에 bind하려면 boolean 설정이 필요합니다.

```bash
# HAProxy가 임의 포트에 bind 허용
setsebool -P haproxy_connect_any 1

# 확인
getsebool haproxy_connect_any
# 기대: haproxy_connect_any --> on
```

이 boolean을 설정하지 않으면 HAProxy가 6443에 bind할 때 `Permission denied`로 실패합니다.

---

## 2.5 Mirror Registry 구성

### 2.5.1 Mirror Registry 선택

OpenShift 폐쇄망 미러링에 사용할 수 있는 레지스트리는 여러 종류가 있습니다.

| 옵션 | 특징 | 권장 |
|---|---|---|
| `mirror-registry` (Red Hat 제공) | Quay 기반 단순화 배포, OpenShift 전용 최적화 | PoC 권장 |
| Quay (full) | 엔터프라이즈 기능, GUI 풍부 | 프로덕션 |
| Harbor | 오픈소스, 멀티 프로젝트 지원 | 멀티 클러스터 환경 |
| Docker Registry | 가장 단순, GUI 없음 | 학습용 |

본 PoC는 `mirror-registry` 사용을 권장합니다. Red Hat에서 OpenShift 폐쇄망 설치 전용으로 제공하며, 단일 명령으로 설치 가능합니다.

### 2.5.2 mirror-registry 설치

```bash
# 작업 디렉토리
mkdir -p /opt/mirror-registry
cd /opt/mirror-registry

# mirror-registry 바이너리 다운로드 (외부망 접근 가능한 시스템에서)
# https://console.redhat.com/openshift/downloads → "mirror registry for Red Hat OpenShift"
# 폐쇄망으로 옮긴 후
tar -xzf mirror-registry.tar.gz

# 설치 (Bastion에 localhost 기반으로 설치)
./mirror-registry install \
  --quayHostname mirror.ocp1.example.com \
  --quayRoot /opt/mirror-registry/quay \
  --quayStorage /opt/mirror-registry/quay-storage \
  --initUser admin \
  --initPassword 'RedHat123!' \
  --sslCert /opt/mirror-registry/tls.crt \
  --sslKey /opt/mirror-registry/tls.key

# 또는 자체 서명 인증서 자동 생성 사용
./mirror-registry install \
  --quayHostname mirror.ocp1.example.com \
  --initUser admin \
  --initPassword 'RedHat123!'
```

설치가 완료되면 다음이 자동 구성됩니다.
- Quay 컨테이너 (Podman으로 실행)
- 자동 생성된 자체 서명 인증서 (또는 사용자 지정)
- 기본 포트: 8443 (HTTPS), 8444 (Quay 콘솔)
- systemd 서비스 등록

### 2.5.3 인증서 신뢰 설정

OpenShift 노드들이 Mirror Registry에서 이미지를 pull하려면 Mirror Registry의 인증서를 신뢰해야 합니다.

```bash
# Mirror Registry 인증서 위치 확인
ls /opt/mirror-registry/quay/quay-rootCA/
# 기대: rootCA.crt rootCA.key rootCA.srl

# Bastion 시스템에 신뢰 추가
cp /opt/mirror-registry/quay/quay-rootCA/rootCA.crt /etc/pki/ca-trust/source/anchors/mirror-registry-ca.crt
update-ca-trust

# 검증
podman login mirror.ocp1.example.com:8443 -u admin -p 'RedHat123!'
# 기대: Login Succeeded!
```

> **install-config.yaml의 additionalTrustBundle**
> OpenShift 노드들도 같은 CA를 신뢰해야 합니다. 이를 위해 install-config.yaml에 `additionalTrustBundle` 항목으로 CA 인증서를 PEM 형식으로 포함시킵니다. 자세한 내용은 Part 3에서 다룹니다.

### 2.5.4 Mirror Registry 검증

```bash
# 로그인 테스트
podman login mirror.ocp1.example.com:8443

# 카탈로그 조회 (초기에는 비어 있음)
curl -k -u admin:RedHat123! https://mirror.ocp1.example.com:8443/v2/_catalog

# 응답 예시 (초기)
# {"repositories":[]}
```

---

## 2.6 NTP/Chrony 위계 설계

### 2.6.1 시간 동기화의 중요성

OpenShift 클러스터는 etcd와 인증서 검증에서 노드 간 시간 동기화에 매우 민감합니다.

- **etcd**: 노드 간 시간 차이가 500ms를 넘으면 election에 영향
- **인증서**: 노드 시간이 발급/유효기간 범위 밖이면 인증 실패
- **로그 분석**: 노드 간 timestamp가 어긋나면 장애 추적이 어려워짐

### 2.6.2 NTP 위계 설계

PoC 환경에서는 다음 위계를 권장합니다.

```
[기업 NTP 서버 또는 외부 NTP]
            ↓
    [Bastion (Stratum 2-3)]
            ↓
[OpenShift 노드 (Stratum 3-4)]
```

- **Bastion**: 외부 NTP에서 시간을 받아 클러스터 노드에 제공
- **OpenShift 노드**: Bastion에서 시간 동기화

폐쇄망에서 외부 NTP 접근이 불가능하면 Bastion 자체를 Stratum 1 기준으로 설정합니다.

### 2.6.3 Bastion Chrony 설정

`/etc/chrony.conf`:

```chrony
# 외부 NTP 사용 시 (사내 NTP가 있다면 그 IP 사용)
server 0.kr.pool.ntp.org iburst
server 1.kr.pool.ntp.org iburst

# 폐쇄망이고 외부 NTP가 없다면 아래 사용 (Bastion 자체가 기준)
# local stratum 8

# 클라이언트(OpenShift 노드)에게 시간 제공 허용
allow 192.168.10.0/24
allow 192.168.20.0/24

# 시간 보정 한도
makestep 1.0 3

# 시스템 클럭에 보정 적용
rtcsync

# 드리프트 파일
driftfile /var/lib/chrony/drift

# 로그
logdir /var/log/chrony
```

```bash
# 서비스 시작
systemctl enable --now chronyd

# 동기화 상태 확인
chronyc tracking
# 기대 출력의 핵심:
#   Stratum         : 3 (or 2)
#   System time     : 0.000XXXXX seconds slow/fast of NTP time

# Peer 상태 확인
chronyc sources -v

# 클라이언트 접속 확인 (노드가 동기화하기 시작하면 표시됨)
chronyc clients
```

### 2.6.4 OpenShift 노드의 Chrony 설정

OpenShift 노드는 MachineConfig를 통해 chrony 설정을 적용합니다. 본 설정은 Part 4(Day-2 MachineConfig)에서 다룹니다. 미리 보기로는 다음 내용이 들어갑니다.

```yaml
# MachineConfig 예시 (Part 4에서 상세)
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-chrony
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,<base64-encoded-chrony-conf>
        mode: 0644
        path: /etc/chrony.conf
```

노드의 chrony.conf 내용:
```chrony
server 192.168.10.10 iburst   # Bastion을 NTP 서버로 지정
makestep 1.0 3
rtcsync
driftfile /var/lib/chrony/drift
```

### 2.6.5 시간 동기화 검증

설치 후 모든 노드에서:

```bash
# 노드에서 직접 (debug pod 또는 oc debug)
oc debug node/master-0 -- chroot /host chronyc tracking
oc debug node/master-0 -- chroot /host chronyc sources

# 모든 노드의 시간 차이 확인 스크립트 예시
for node in master-0 master-1 master-2 ap-worker-0 ap-worker-1 \
            db-worker-0 db-worker-1 podpool1-worker-0 podpool1-worker-1; do
  echo "=== $node ==="
  oc debug node/$node -- chroot /host date +"%Y-%m-%d %H:%M:%S.%3N"
done
```

모든 노드의 시간이 100ms 이내로 일치해야 정상입니다.

---

## 2.7 Web Server (Ignition 제공)

### 2.7.1 Ignition 파일 제공의 역할

UPI 설치 방식에서는 OpenShift 노드가 부팅할 때 ignition 파일을 HTTP로 가져갑니다. Bastion이 이 ignition 파일들을 호스팅합니다.

| Ignition 파일 | 용도 |
|---|---|
| `bootstrap.ign` | Bootstrap 노드용 |
| `master.ign` | 모든 Master 노드용 (동일 파일) |
| `worker.ign` | 모든 Worker 노드용 (ap/db/podpool1 모두 동일) |

> **Worker ignition은 모두 같다**
> ap, db, podpool1은 처음에는 모두 `worker.ign`으로 부팅됩니다. MCP 분리(ap/db/podpool1)는 설치 후 노드 라벨링을 통해 이루어지며, 이때 각 MCP에 맞는 MachineConfig가 적용되어 노드가 재구성됩니다. (Part 4 참조)

### 2.7.2 httpd 구성

```bash
# httpd 설정
systemctl enable --now httpd

# ignition 파일 디렉토리
mkdir -p /var/www/html/ignition
mkdir -p /var/www/html/rhcos

# SELinux context 적용
chcon -t httpd_sys_content_t /var/www/html/ignition -R
chcon -t httpd_sys_content_t /var/www/html/rhcos -R

# httpd 기본 포트 변경 (8080 권장, 80은 다른 용도 가능성)
sed -i 's/^Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf

# SELinux: httpd가 8080 포트 사용 허용
semanage port -a -t http_port_t -p tcp 8080 || true

systemctl restart httpd

# 검증
curl http://192.168.10.10:8080/
```

### 2.7.3 RHCOS 이미지 호스팅

RHCOS Live ISO와 disk image도 Bastion이 제공합니다. (RHCOS 다운로드와 노드 설치 절차는 Part 3에서 다룹니다.)

```bash
# RHCOS 이미지 배치 위치
ls /var/www/html/rhcos/
# rhcos-4.20.x-x86_64-live.x86_64.iso
# rhcos-4.20.x-x86_64-live-kernel-x86_64
# rhcos-4.20.x-x86_64-live-initramfs.x86_64.img
# rhcos-4.20.x-x86_64-live-rootfs.x86_64.img
```

---

## 2.8 oc 및 openshift-install 도구 배치

### 2.8.1 도구 다운로드

폐쇄망이므로 외부망 시스템에서 다운로드 후 Bastion으로 옮겨야 합니다.

```bash
# Bastion에 도구 배치 디렉토리
mkdir -p /opt/openshift/bin
cd /opt/openshift/bin

# 다음 파일들을 외부망에서 다운로드하여 옮김:
# - oc-4.20.x-linux.tar.gz
# - openshift-install-4.20.x-linux.tar.gz
# - oc-mirror.tar.gz

# 압축 해제
tar -xzf oc-4.20.x-linux.tar.gz
tar -xzf openshift-install-4.20.x-linux.tar.gz
tar -xzf oc-mirror.tar.gz

# PATH에 추가
ln -sf /opt/openshift/bin/oc /usr/local/bin/oc
ln -sf /opt/openshift/bin/kubectl /usr/local/bin/kubectl
ln -sf /opt/openshift/bin/openshift-install /usr/local/bin/openshift-install
ln -sf /opt/openshift/bin/oc-mirror /usr/local/bin/oc-mirror

# bash completion
oc completion bash > /etc/bash_completion.d/oc

# 버전 확인
oc version --client
openshift-install version
oc-mirror version
```

### 2.8.2 도구 버전 일치의 중요성

- `oc`와 `openshift-install`은 OpenShift 클러스터 버전과 일치하거나 호환 가능한 버전이어야 합니다.
- `oc-mirror`는 v2 사용. v1과 명령어와 출력물 구조가 다릅니다.

> **버전 불일치로 인한 흔한 문제**
> - 4.20 클러스터에 4.18 oc로 접근: 일부 신규 API 미지원
> - 4.18 openshift-install로 4.20 install-config 생성: 일부 필드 검증 실패
> - oc-mirror v1과 v2 혼용: ImageContentSourcePolicy(v1)과 ImageDigestMirrorSet(v2) 충돌
>
> **반드시 4.20 버전 도구로 통일하십시오.**

---

## 2.9 vBMC 구성 (FAR 테스트용)

### 2.9.1 vBMC의 역할

PoC 환경에서는 모든 OpenShift 노드가 VM이므로 물리 BMC(IPMI/iDRAC/iLO)가 없습니다. FAR(Fence Agents Remediation) 동작을 검증하려면 BMC를 에뮬레이션할 수 있는 도구가 필요한데, 이를 위해 `virtualbmc`(vBMC)를 사용합니다.

vBMC는 다음을 수행합니다.
- VM에 BMC endpoint를 매핑
- IPMI 명령(`ipmitool`)을 받아 hypervisor API(libvirt)로 전환
- VM의 전원 제어 (power on/off/reset)

상세 구성은 Part 6 (FAR/vBMC 테스트 시나리오)에서 다룹니다. 본 파트에서는 Bastion에 vBMC 설치만 미리 준비합니다.

### 2.9.2 vBMC 패키지 준비

```bash
# vBMC는 Python 패키지
dnf install -y python3 python3-pip ipmitool

# 외부망에서 다운로드한 vBMC wheel 파일을 Bastion으로 옮긴 후
pip3 install --no-index --find-links=/opt/vbmc-packages virtualbmc

# 설치 확인
vbmc --version
ipmitool -V
```

vBMC 데몬 시작과 노드별 BMC 등록은 Part 6에서 다룹니다.

---

## 2.10 Bastion 통합 검증

Bastion 모든 구성 요소가 준비되었다면 통합 검증을 수행합니다.

### 2.10.1 서비스 상태 확인

```bash
# 모든 핵심 서비스 상태
systemctl status named haproxy httpd chronyd | grep -E "Active:|●"
```

기대 출력: 모두 `active (running)`

### 2.10.2 포트 listen 확인

```bash
ss -tlnp | grep -E ':(53|80|443|6443|8080|8443|22623|9000)'
```

기대 listen 포트:

| 포트 | 서비스 | bind IP |
|---|---|---|
| 53 | named (DNS) | 192.168.10.10 |
| 80 | HAProxy (Default Ingress HTTP) | 192.168.10.21 |
| 443 | HAProxy (Default Ingress HTTPS) | 192.168.10.21 |
| 80 | HAProxy (Service Ingress HTTP) | 192.168.20.20 |
| 443 | HAProxy (Service Ingress HTTPS) | 192.168.20.20 |
| 6443 | HAProxy (API) | 192.168.10.20 |
| 22623 | HAProxy (MCS) | 192.168.10.20 |
| 8080 | httpd (Ignition) | 0.0.0.0 |
| 8443 | mirror-registry | 0.0.0.0 |
| 9000 | HAProxy stats | 192.168.10.10 |

### 2.10.3 DNS 통합 테스트

```bash
# 정방향
for name in api api-int console-openshift-console.apps oauth-openshift.apps frontend.svcapps \
            bootstrap master-0 master-1 master-2 \
            ap-worker-0 ap-worker-1 db-worker-0 db-worker-1 \
            podpool1-worker-0 podpool1-worker-1 mirror; do
  printf "%-50s → " "$name.ocp1.example.com"
  dig @192.168.10.10 $name.ocp1.example.com +short
done

# 역방향
for ip in 10 20 21 30 31 32 33 41 42 51 52 61 62 70; do
  printf "192.168.10.%-3s → " "$ip"
  dig @192.168.10.10 -x 192.168.10.$ip +short
done
```

### 2.10.4 HAProxy 헬스체크 확인

```bash
# HAProxy 통계 페이지를 통한 backend 상태 확인
curl -s -u admin:RedHat123! 'http://192.168.10.10:9000/;csv' | \
  awk -F, '{print $1, $2, $18}' | column -t
```

설치 전이라 모든 backend가 DOWN인 것이 정상입니다.

### 2.10.5 Mirror Registry 검증

```bash
# 로그인
podman login -u admin -p 'RedHat123!' mirror.ocp1.example.com:8443

# 헬스체크
curl -k https://mirror.ocp1.example.com:8443/health/instance
# 기대: {"data":{"services":...},"status_code":200}
```

### 2.10.6 시간 동기화 확인

```bash
chronyc tracking
chronyc sources -v

# Stratum이 표시되고, Reach가 모두 채워져 있어야 함
```

---

## 2.11 Part 2 학습 점검

다음 질문에 답할 수 있다면 Part 2를 충분히 학습한 것입니다.

1. Bastion이 OpenShift 설치/운영 라이프사이클의 어느 시점에 어떤 서비스를 제공하는가? (서비스별로 답하시오)
2. `api`와 `api-int` DNS 레코드가 분리되어 있는 이유는 무엇인가?
3. HAProxy의 backend 헬스체크에서 TCP 443 단순 체크 대신 1936/healthz를 사용하는 이유는?
4. HAProxy를 `mode tcp`로 설정하는 이유는 무엇인가? `mode http`를 쓰면 어떻게 되는가?
5. PodPool1 노드의 Service IP를 DNS에 등록하지 않는 이유는?
6. Mirror Registry의 CA 인증서를 OpenShift 노드들이 신뢰하게 만드는 방법은?
7. Bastion이 자체 NTP 기준이 되는 경우와 외부 NTP를 사용하는 경우의 chrony.conf 차이는?
8. bootstrap 노드가 LB의 backend에 있다가 설치 완료 후 제거되어야 하는 이유는?
9. SELinux에서 `haproxy_connect_any` boolean을 활성화해야 하는 이유는?
10. PoC에서 firewalld를 비활성화하는 결정의 trade-off는?

---

*Part 2 끝. 다음은 Part 3 (폐쇄망 미러링 및 OpenShift 설치) 입니다.*
