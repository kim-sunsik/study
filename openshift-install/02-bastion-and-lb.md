# Part 2. Bastion 및 Load Balancer 구성

> **목적**  
> Bastion 서버에 DNS, HAProxy Load Balancer, Mirror Registry, Ignition 제공 Web Server, NTP/Chrony 중계 기능을 구성합니다. PoC에서는 Bastion이 Infra망과 Service망을 모두 가지며, API/MCS, Default Ingress, Service Ingress의 VIP를 처리합니다.

---

## 2.1 Bastion 역할과 네트워크

### 2.1.1 역할

| 기능 | 설명 |
|---|---|
| DNS | `api`, `api-int`, `*.apps`, `*.svcapps`, 노드 FQDN 제공 |
| HAProxy LB | API/MCS, Default Ingress, Service Ingress 부하분산 |
| Mirror Registry | 폐쇄망 이미지 저장소 |
| Web Server | Ignition 파일과 RHCOS 이미지 제공 |
| NTP/Chrony | 폐쇄망 시간 동기화 기준 또는 중계 |
| vBMC 관리 | FAR fencing PoC용 가상 BMC 관리 |
| 설치 도구 | `oc`, `openshift-install`, `oc-mirror`, `coreos-installer` 준비 |

---

### 2.1.2 Bastion IP/VIP 설계

```text
Bastion
├─ Infra NIC: 192.168.10.10/24
│  ├─ VIP 192.168.10.20  # API / MCS
│  └─ VIP 192.168.10.21  # Default Ingress
└─ Service NIC: 192.168.20.10/24
   └─ VIP 192.168.20.20  # Service Ingress
```

| 항목 | IP | 용도 |
|---|---|---|
| Bastion Infra IP | `192.168.10.10` | DNS, Mirror, SSH, Web |
| API/MCS VIP | `192.168.10.20` | API 6443, MCS 22623 |
| Default Ingress VIP | `192.168.10.21` | 관리용 apps |
| Bastion Service IP | `192.168.20.10` | Service망 HAProxy bind 용 |
| Service Ingress VIP | `192.168.20.20` | 업무용 svcapps |

> **PoC vs 프로덕션**  
> PoC에서는 Bastion이 Service Ingress VIP까지 처리합니다. 프로덕션에서는 `192.168.20.20`을 별도 물리 L4/L7로 이전하는 것을 권장합니다.

---

## 2.2 Bastion OS 기본 설정

### 2.2.1 호스트명 설정

```bash
sudo hostnamectl set-hostname bastion.ocp420.aonsoft.demo.com
```

### 2.2.2 필수 패키지 설치

```bash
sudo dnf install -y \
  bind bind-utils \
  haproxy \
  httpd \
  chrony \
  podman skopeo jq \
  tar gzip unzip vim bash-completion \
  firewalld

sudo systemctl enable --now firewalld
```

---

## 2.3 Bastion 네트워크 구성 예시

> NIC 이름은 환경마다 다릅니다. 아래 예시는 `ens192=Infra`, `ens224=Service`를 가정합니다.

### 2.3.1 Infra NIC

```bash
sudo nmcli con mod ens192 ipv4.method manual \
  ipv4.addresses 192.168.10.10/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns 192.168.10.10 \
  ipv4.dns-search ocp420.aonsoft.demo.com \
  connection.autoconnect yes
```

### 2.3.2 Service NIC

```bash
sudo nmcli con mod ens224 ipv4.method manual \
  ipv4.addresses 192.168.20.10/24 \
  ipv4.never-default yes \
  connection.autoconnect yes
```

### 2.3.3 VIP 추가

```bash
sudo nmcli con mod ens192 +ipv4.addresses 192.168.10.20/24
sudo nmcli con mod ens192 +ipv4.addresses 192.168.10.21/24
sudo nmcli con mod ens224 +ipv4.addresses 192.168.20.20/24
sudo nmcli con up ens192
sudo nmcli con up ens224
```

검증:

```bash
ip addr show ens192
ip addr show ens224
ip route show default
```

기대값:

```text
default via 192.168.10.1 dev ens192
192.168.20.0/24 dev ens224
```

---

## 2.4 DNS 구성

### 2.4.1 named 설정

`/etc/named.conf`에 zone을 추가합니다.

```conf
options {
    listen-on port 53 { 127.0.0.1; 192.168.10.10; };
    listen-on-v6 port 53 { none; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file   "/var/named/data/named.secroots";
    recursing-file  "/var/named/data/named.recursing";
    allow-query     { any; };
    recursion yes;
    dnssec-validation no;
};

zone "ocp420.aonsoft.demo.com" IN {
    type master;
    file "ocp420.aonsoft.demo.com.zone";
    allow-update { none; };
};
```

### 2.4.2 Zone 파일

`/var/named/ocp420.aonsoft.demo.com.zone`

```dns
$TTL 1D
@   IN SOA  bastion.ocp420.aonsoft.demo.com. admin.aonsoft.demo.com. (
        2026051501 ; serial
        1H         ; refresh
        15M        ; retry
        1W         ; expire
        1D )       ; minimum

    IN NS   bastion.ocp420.aonsoft.demo.com.

bastion                         IN A 192.168.10.10
mirror                          IN A 192.168.10.10
nas                             IN A 192.168.10.70

api                             IN A 192.168.10.20
api-int                         IN A 192.168.10.20
*.apps                          IN A 192.168.10.21
*.svcapps                       IN A 192.168.20.20

bootstrap                       IN A 192.168.10.30
master-0                        IN A 192.168.10.31
master-1                        IN A 192.168.10.32
master-2                        IN A 192.168.10.33
ap-worker-0                     IN A 192.168.10.41
ap-worker-1                     IN A 192.168.10.42
db-worker-0                     IN A 192.168.10.51
db-worker-1                     IN A 192.168.10.52
podpool1-worker-0               IN A 192.168.10.61
podpool1-worker-1               IN A 192.168.10.62
```

### 2.4.3 권한 및 서비스 기동

```bash
sudo chown root:named /var/named/ocp420.aonsoft.demo.com.zone
sudo restorecon -Rv /var/named
sudo named-checkconf
sudo named-checkzone ocp420.aonsoft.demo.com /var/named/ocp420.aonsoft.demo.com.zone
sudo systemctl enable --now named
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
```

### 2.4.4 DNS 검증

```bash
dig @192.168.10.10 api.ocp420.aonsoft.demo.com +short
dig @192.168.10.10 api-int.ocp420.aonsoft.demo.com +short
dig @192.168.10.10 console-openshift-console.apps.ocp420.aonsoft.demo.com +short
dig @192.168.10.10 sample.svcapps.ocp420.aonsoft.demo.com +short
```

기대값:

```text
api      → 192.168.10.20
api-int  → 192.168.10.20
*.apps   → 192.168.10.21
*.svcapps → 192.168.20.20
```

---

## 2.5 HAProxy 구성

### 2.5.1 방화벽 포트

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=22623/tcp
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

### 2.5.2 HAProxy 설정

`/etc/haproxy/haproxy.cfg`

```haproxy
global
  log /dev/log local0
  log /dev/log local1 notice
  daemon
  maxconn 20000

defaults
  log global
  mode tcp
  option tcplog
  option dontlognull
  timeout connect 10s
  timeout client  5m
  timeout server  5m

# ------------------------------------------------------------
# Kubernetes API Server
# ------------------------------------------------------------
frontend api_6443
  bind 192.168.10.20:6443
  mode tcp
  default_backend api_backend

backend api_backend
  mode tcp
  balance roundrobin
  option tcp-check
  server bootstrap 192.168.10.30:6443 check
  server master-0 192.168.10.31:6443 check
  server master-1 192.168.10.32:6443 check
  server master-2 192.168.10.33:6443 check

# ------------------------------------------------------------
# Machine Config Server
# ------------------------------------------------------------
frontend mcs_22623
  bind 192.168.10.20:22623
  mode tcp
  default_backend mcs_backend

backend mcs_backend
  mode tcp
  balance roundrobin
  option tcp-check
  server bootstrap 192.168.10.30:22623 check
  server master-0 192.168.10.31:22623 check
  server master-1 192.168.10.32:22623 check
  server master-2 192.168.10.33:22623 check

# ------------------------------------------------------------
# Default Ingress - Infra Apps
# ------------------------------------------------------------
frontend infra_ingress_80
  bind 192.168.10.21:80
  mode tcp
  default_backend default_router_80

frontend infra_ingress_443
  bind 192.168.10.21:443
  mode tcp
  default_backend default_router_443

backend default_router_80
  mode tcp
  balance roundrobin
  server ap0 192.168.10.41:80 check port 1936
  server ap1 192.168.10.42:80 check port 1936

backend default_router_443
  mode tcp
  balance roundrobin
  server ap0 192.168.10.41:443 check port 1936
  server ap1 192.168.10.42:443 check port 1936

# ------------------------------------------------------------
# Service Ingress - Business Apps
# ------------------------------------------------------------
frontend service_ingress_80
  bind 192.168.20.20:80
  mode tcp
  default_backend service_router_80

frontend service_ingress_443
  bind 192.168.20.20:443
  mode tcp
  default_backend service_router_443

backend service_router_80
  mode tcp
  balance roundrobin
  server podpool0 192.168.20.61:80 check port 1936
  server podpool1 192.168.20.62:80 check port 1936

backend service_router_443
  mode tcp
  balance roundrobin
  server podpool0 192.168.20.61:443 check port 1936
  server podpool1 192.168.20.62:443 check port 1936
```

### 2.5.3 HAProxy 기동 및 검증

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable --now haproxy
sudo systemctl status haproxy
ss -tlnp | grep haproxy
```

VIP bind 확인:

```bash
ss -tlnp | grep -E ':(6443|22623|80|443)'
```

---

## 2.6 Bootstrap 완료 후 HAProxy 정리

Bootstrap 완료 전에는 API/MCS backend에 bootstrap이 포함되어야 합니다. 완료 후 반드시 제거합니다.

제거 전:

```haproxy
server bootstrap 192.168.10.30:6443 check
server bootstrap 192.168.10.30:22623 check
```

제거 후:

```haproxy
backend api_backend
  mode tcp
  balance roundrobin
  option tcp-check
  server master-0 192.168.10.31:6443 check
  server master-1 192.168.10.32:6443 check
  server master-2 192.168.10.33:6443 check

backend mcs_backend
  mode tcp
  balance roundrobin
  option tcp-check
  server master-0 192.168.10.31:22623 check
  server master-1 192.168.10.32:22623 check
  server master-2 192.168.10.33:22623 check
```

적용:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

---

## 2.7 Ignition Web Server 구성

```bash
sudo dnf install -y httpd
sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo mkdir -p /var/www/html/ocp420
```

Ignition 파일 생성 후 다음 위치에 복사합니다.

```bash
sudo cp ~/ocp420-install/*.ign /var/www/html/ocp420/
sudo chmod 644 /var/www/html/ocp420/*.ign
```

검증:

```bash
curl http://192.168.10.10/ocp420/bootstrap.ign
curl http://192.168.10.10/ocp420/master.ign
curl http://192.168.10.10/ocp420/worker.ign
```

---

## 2.8 Chrony/NTP 구성

### 2.8.1 Bastion chrony 서버

`/etc/chrony.conf` 예시:

```conf
# 외부 NTP가 없다면 PoC 기준 local stratum 사용
local stratum 10

allow 192.168.10.0/24
allow 192.168.20.0/24

makestep 1.0 3
rtcsync
logdir /var/log/chrony
```

기동:

```bash
sudo systemctl enable --now chronyd
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --reload
chronyc tracking
chronyc sources -v
```

> **프로덕션 권장**  
> Bastion 자체 시간을 기준으로 삼는 구성은 PoC에 한정합니다. 운영 환경에서는 사내 표준 NTP, GPS, 또는 상위 Stratum 서버와 연계하는 것을 권장합니다.

---

## 2.9 Router health check 주의

HAProxy backend에서 `check port 1936`을 사용합니다. 이를 위해 다음 통신이 허용되어야 합니다.

| 방향 | 포트 | 용도 |
|---|---:|---|
| Bastion → ap 노드 | 1936/TCP | default router health check |
| Bastion → podpool1 Service IP | 1936/TCP | service router health check |

검증:

```bash
nc -vz 192.168.10.41 1936
nc -vz 192.168.10.42 1936
nc -vz 192.168.20.61 1936
nc -vz 192.168.20.62 1936
```

> 설치 직후에는 router Pod가 아직 없으므로 health check가 실패할 수 있습니다. IngressController 배치 이후 다시 확인합니다.

---

## 2.10 실패 시 확인

| 증상 | 확인 항목 |
|---|---|
| `api` 이름 해석 실패 | DNS zone, named 상태, 방화벽 53 |
| API 6443 접속 실패 | HAProxy bind, master backend, VIP 존재 여부 |
| Bootstrap 완료 지연 | MCS 22623, bootstrap backend, ignition URL |
| Console 접속 실패 | `*.apps` DNS, default router 위치, ap backend |
| 업무 Route 접속 실패 | `*.svcapps` DNS, service router 위치, podpool Service IP |
| HAProxy 기동 실패 | `haproxy -c`, VIP bind 충돌, SELinux/방화벽 |
| Router health check 실패 | 1936 포트, router Pod 상태, nodePlacement |

---

## 2.11 PoC vs 프로덕션 차이

| 항목 | PoC | 프로덕션 권장 |
|---|---|---|
| DNS | Bastion named | 사내 DNS 또는 이중화 DNS |
| API/MCS LB | Bastion HAProxy | 관리망 L4 또는 HAProxy 이중화 |
| Default Ingress LB | Bastion HAProxy | 관리망 L4 |
| Service Ingress LB | Bastion HAProxy | 별도 물리 L4/L7 |
| Bastion NIC | Infra + Service | Infra 중심, Service LB 분리 |
| Chrony | Bastion local stratum 가능 | 사내 표준 NTP 연계 |

---

## 2.12 Part 2 학습 점검

1. API VIP와 Default Ingress VIP를 분리하는 이유는 무엇인가?
2. PoC에서 Bastion이 Service망 NIC를 가져야 하는 이유는 무엇인가?
3. Bootstrap 완료 후 HAProxy에서 제거해야 하는 backend는 무엇인가?
4. Default Ingress와 Service Ingress의 backend 노드는 각각 어디인가?
5. Router health check에 1936 포트를 사용하는 이유는 무엇인가?

---

*Part 2 끝. 다음은 Part 3 — 폐쇄망 미러링 및 UPI 설치입니다.*
