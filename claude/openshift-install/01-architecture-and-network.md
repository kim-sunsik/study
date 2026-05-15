# Part 1. 아키텍처 및 네트워크 설계

> **이 문서의 목적**
> 본 가이드는 폐쇄망 환경에서 OpenShift Container Platform 4.20을 UPI 방식으로 설치하고, VM 워크로드와 컨테이너 워크로드를 분리 운영하며, MTV(Migration Toolkit for Virtualization) 및 FAR(Fence Agents Remediation) 시나리오를 검증하는 PoC를 수행하기 위한 교육자료입니다.
>
> 단순한 절차 모음이 아니라 "왜 이렇게 설계하는가"를 학습할 수 있도록 각 결정의 배경, 대안, 흔한 실수를 함께 다룹니다.

---

## 1.1 문서 범위와 학습 목표

### 1.1.1 다루는 환경

본 문서는 다음 환경을 전제로 합니다.

- **OpenShift Container Platform 4.20**
- **폐쇄망(Disconnected) 환경** — 클러스터 노드가 인터넷에 직접 접근 불가
- **UPI(User Provisioned Infrastructure)** 설치 방식 — 사용자가 노드/네트워크/LB를 직접 준비
- **Bastion 단일 노드 기반** — DNS, Load Balancer, Mirror Registry 통합
- **oc-mirror v2** 기반 이미지 미러링
- **VM 전용 Worker Pool과 Container 전용 Worker Pool 분리 운영**
- **OpenShift Virtualization** 설치 및 VM 워크로드 실행
- **MTV** 기반 가상 머신 마이그레이션 테스트
- **NAS 기반 StorageClass** — PoC 환경에서는 Infra망과 통합
- **FAR + vBMC** 기반 fencing 테스트

### 1.1.2 핵심 설계 원칙

설계 전체를 관통하는 다섯 가지 원칙입니다. 이후 모든 의사결정은 이 원칙에서 파생됩니다.

1. **관리 트래픽은 Infra망, 서비스 트래픽은 Service망** — 트래픽 종류와 네트워크가 1:1로 대응
2. **Master는 control plane 전용** — Service망에 연결하지 않음
3. **VM 노드와 컨테이너 노드는 분리** — 워크로드 격리와 자원 경합 회피
4. **PoC와 프로덕션 구성을 문서에서 명시적으로 분리** — 전환 시 변경점이 자동으로 추적됨
5. **NAS는 PoC에서는 Infra망과 통합, 프로덕션에서는 분리 검토** — 자원 제약과 분리 원칙의 균형

### 1.1.3 학습 목표

본 문서를 학습한 후 학습자는 다음을 수행할 수 있어야 합니다.

- OpenShift UPI 설치의 네트워크 설계를 직접 그릴 수 있다.
- `machineNetwork`, `clusterNetwork`, `serviceNetwork`의 차이를 설명할 수 있다.
- 두 IngressController를 분리 운영하는 이유와 방법을 설명할 수 있다.
- 노드별 NIC/Bond 구성을 트래픽 흐름과 연결지어 설계할 수 있다.
- PoC 설계와 프로덕션 설계의 차이를 식별할 수 있다.

---

## 1.2 전체 아키텍처

### 1.2.1 노드 구성

PoC 환경의 노드 구성은 다음과 같습니다.

| 역할 | 수량 | MCP | 주요 용도 |
|---|---|---|---|
| Bastion | 1 | (해당 없음) | DNS, LB, Mirror Registry, Ignition 제공, vBMC, NTP |
| Master | 3 | `master` | API 서버, etcd, scheduler, controller |
| VM Worker (AP) | 2 | `ap` | AP성 VM, Default IngressController |
| VM Worker (DB) | 2 | `db` | DB성 VM |
| Container Worker (PodPool1) | 2 | `podpool1` | 컨테이너 업무 워크로드, Service IngressController |

> **왜 ap와 db를 분리하는가**
> ap와 db는 둘 다 VM 전용이고 NIC 구성도 동일하지만, 다음 이유로 MCP를 분리합니다.
> - VM 업무군별 장애 영향 범위 분리
> - 향후 CPU pinning / hugepages / reserved CPU 정책 차등 적용
> - kdump/crashkernel 설정 차등 적용
> - AP성 VM과 DB성 VM의 운영 패치 윈도우 분리
>
> 운영 단순성이 우선이라면 ap/db를 단일 `vm-worker` MCP로 통합할 수 있습니다. 본 PoC는 운영 정책 분리 검증을 포함하므로 ap/db를 분리합니다.

### 1.2.2 전체 구성도

```
                  [ 외부 관리자 ]         [ 외부 사용자 (컨테이너 앱) ]    [ 외부 사용자 (VM 서비스) ]
                       │                       │                              │
                       │                       │                              │ (Bastion 미경유)
                       │                       │                              │ NAD로 L2 직결
                       ▼                       ▼                              │
            ┌────────────────────────────────────────────┐                    │
            │  Bastion HAProxy (단일 프로세스, 논리 분리) │                    │
            │  ─────────────────────────────────────────  │                    │
            │  [Infra 측 frontend]                        │                    │
            │   - API/MCS    .10.20:6443 / :22623         │                    │
            │   - Default Ingress  .10.21:80/443          │                    │
            │  [Service 측 frontend]                      │                    │
            │   - Service Ingress  .20.20:80/443          │                    │
            └────────────────────────────────────────────┘                    │
                       │                       │                              │
              ┌────────┴──────────┐            │                              │
              │                   │            │                              │
              ▼                   ▼            ▼                              │
         [Master x3]          [AP x2]      [PodPool1 x2]                      │
         Infra NIC만           Infra NIC    Infra NIC                         │
         (관리 전용)           (호스트)     + Service NIC                     │
              │                + ens224     (호스트 IP 양쪽)                  │
              │                  ↓          → service router (Host            │
              │                br-vm-svc      Network 80/443)                 │
              │                bridge                                         │
              │                (IP 없음,                                      │
              │                 L2 통로)                                      │
              │                  │           [DB x2]                          │
              │                  │           (AP와 동일 NIC 구성)             │
              │                  │            │                               │
              │                  ▼            ▼                               │
              │                ┌──── VM Guest OS ────┐ ◄────────────────────┘
              │                │ Service IP 직접 보유 │  (외부 ↔ VM 직결)
              │                │ 192.168.20.100~199  │
              │                └─────────────────────┘

  ─── Infra + NAS Network (192.168.10.0/24): 모든 노드 호스트 IP, 관리 트래픽, NAS ───
  ─── Service Network    (192.168.20.0/24): podpool1 호스트 IP + VM Guest IP        ───
```

**그림에서 읽어야 할 핵심:**

1. **관리 트래픽 (Bastion 경유)**
   외부 관리자 → Bastion Infra VIP(.10.20/.10.21) → master/ap router

2. **컨테이너 업무 트래픽 (Bastion 경유)**
   외부 사용자 → Bastion Service VIP(.20.20) → podpool1 service router → 업무 Pod

3. **VM 서비스 트래픽 (Bastion 미경유, L2 직결)**
   외부 사용자 → Service망 스위치 → ap/db의 ens224 (bridge port) → br-vm-svc → VM Guest
   - VM은 Service망의 일원으로 동작하며, IP를 직접 보유 (192.168.20.100~199)
   - ap/db 호스트는 Service NIC에 IP가 없고 L2 통로 역할만 함
   - Bastion HAProxy의 Service Ingress VIP(20.20)는 **컨테이너 업무 전용**이며 VM과 무관

### 1.2.3 트래픽 흐름 요약

각 트래픽이 어느 망을 통해 어떻게 흐르는지 한눈에 보기 위한 매트릭스입니다. **경유 장비 컬럼**을 추가하여 Bastion HAProxy를 거치는 트래픽과 거치지 않는 트래픽을 명확히 구분합니다.

| 트래픽 종류 | 방향 | 망 | 경유 장비 | 진입/출구 메커니즘 |
|---|---|---|---|---|
| API / kubelet / etcd / MCS | 노드↔노드 | Infra | (직접) | machineNetwork 내부 통신 |
| API 외부 호출 | 외부→API | Infra | **Bastion HAProxy** | API VIP(.10.20):6443 |
| MCS (설치 시) | 노드→MCS | Infra | **Bastion HAProxy** | MCS VIP(.10.20):22623 |
| 이미지 pull | 노드→Mirror | Infra | (직접) | default gateway 경유 |
| Console / OAuth | 외부→내부 | Infra | **Bastion HAProxy** | Default IngressController (ap 노드) |
| Monitoring outbound | 내부→외부 | Infra | (직접) | default gateway 경유 |
| 업무 앱 inbound | 외부→내부 | Service | **Bastion HAProxy** | Service IngressController (podpool1 노드) |
| 업무 앱 outbound | 내부→외부 | Service | (직접) | EgressIP 또는 정책 라우팅 (PoC 검증 항목) |
| **VM inbound** | **외부→VM** | **Service** | **(Bastion 미경유, L2 직결)** | **NAD bridge로 VM Guest IP에 직접 도달** |
| **VM outbound** | **VM→외부** | **Service** | **(Bastion 미경유, L2 직결)** | **VM Guest OS의 default gateway = 192.168.20.1** |
| NAS storage | 노드↔NAS | Infra (PoC) | (직접) | machineNetwork 내부 |

> **교육 포인트**
> 운영자 입장에서 "이 트래픽은 어느 망으로 가야 하지?"를 고민하는 것이 아니라, **트래픽의 성격(관리 vs 서비스)이 결정되면 망은 자동으로 결정**되도록 설계하는 것이 핵심입니다.

> **자주 묻는 질문: VM은 외부에서 어떻게 접근하는가?**
> VM은 Bastion HAProxy를 경유하지 않습니다. Service망의 L2 멤버로서 외부와 직접 통신합니다. 접근 방식은 다음 중에서 환경에 맞춰 선택합니다.
>
> | 방식 | 설명 | 본 PoC 적용 |
> |---|---|---|
> | **VM Guest IP 직접 접근** | 외부 사용자가 192.168.20.150 같은 VM IP에 직접 접근 (SSH, 웹 등) | PoC 기본 |
> | **VM DNS 등록** | DNS에 VM hostname을 VM Guest IP로 등록 (예: `app01.vm.ocp1.example.com → 192.168.20.150`) | 권장 |
> | **별도 VM용 LB** | VM 다수 앞에 별도 LB(HAProxy/F5 등)를 두는 경우 (Bastion과는 무관) | 운영 시 검토 |
> | **MetalLB** | OpenShift에서 LoadBalancer 타입 Service로 VM에 외부 IP 부여 | 본 PoC 범위 밖 |
>
> 핵심은 **Bastion HAProxy의 Service Ingress VIP(192.168.20.20)는 컨테이너 업무 Route 전용**이라는 점입니다. VM 트래픽은 이 VIP와 완전히 별개의 경로로 흐릅니다.

---

## 1.3 네트워크 설계

### 1.3.1 네트워크 분리 원칙

본 PoC는 두 개의 물리/논리 네트워크를 사용합니다.

**Infra + NAS Network (192.168.10.0/24)**
- 용도: OpenShift 관리 트래픽 + 스토리지
- 포함되는 트래픽
  - Kubernetes API 서버 통신
  - Machine Config Server (MCS) 통신
  - kubelet ↔ API 서버 통신
  - etcd peer 통신
  - DNS 조회
  - Mirror Registry 이미지 pull
  - NAS 스토리지 액세스 (NFS)
  - NTP/Chrony 동기화
  - OpenShift Console / OAuth 접근

**Service Network (192.168.20.0/24)**
- 용도: 사용자 서비스 트래픽
- 포함되는 트래픽
  - 컨테이너 업무 애플리케이션 Ingress
  - 컨테이너 업무 애플리케이션 Egress
  - VM 서비스 트래픽 (외부 ↔ VM)
  - Service Mesh 외부 노출 트래픽

> **NAS를 Infra망에 통합한 이유 (PoC 한정)**
> 운영 환경이라면 NAS는 별도 망(예: 192.168.30.0/24)으로 분리하는 것이 정석입니다. NFS는 트래픽이 크고 latency에 민감하므로 관리망과 경합하면 etcd 안정성에 영향을 줄 수 있습니다.
> 본 PoC는 자원 제약상 NAS와 Infra를 통합하되, 운영 전환 시 NAS 분리를 권장 사항으로 명시합니다.

### 1.3.2 두 사이클 라우팅

설계의 핵심은 **관리 사이클과 서비스 사이클이 각각 한 망 안에서 닫히도록** 만드는 것입니다.

**관리 사이클**
```
관리자 → Infra LB → Default IngressController (ap 노드 Infra NIC)
       → console/oauth Pod
       → 응답: ap 노드 default gateway(Infra) → Infra LB → 관리자
```

이 사이클은 노드의 default gateway가 Infra에 있어 inbound와 outbound가 모두 Infra망 안에서 닫힙니다. **라우팅이 대칭(symmetric routing)이며 별도 설정이 필요 없습니다.**

**서비스 사이클**
```
사용자 → Service LB → Service IngressController (podpool1 Service NIC)
       → 업무 Pod
       → 응답: connection tracking으로 Service NIC 반환 → Service LB → 사용자
```

이 사이클은 inbound는 Service NIC를 통해 들어오지만, 응답이 같은 NIC로 돌아갈 수 있는 이유는 connection tracking 덕분입니다. **단, 업무 Pod가 새로 outbound 연결을 만들 때는 default gateway(Infra)를 따라가므로, Service망으로 outbound를 강제하려면 EgressIP 또는 정책 라우팅이 추가로 필요합니다.**

> **흔한 실수: 비대칭 라우팅**
> Service망에 default gateway를 두면 outbound가 Service망으로 나가지만, kubelet → API 서버 통신 같은 관리 트래픽도 Service망으로 흐르려 시도합니다. machineNetwork에 명시되지 않은 망이므로 ovn-k node IP 선정이 흔들리고, 결과적으로 노드가 NotReady가 되거나 CSR 자동 승인이 실패합니다.
>
> **모든 노드의 default gateway는 반드시 Infra망에 두십시오.**

### 1.3.3 Kubernetes 내부 네트워크

위의 외부 네트워크와 별개로, 클러스터 내부에서 동작하는 가상 네트워크가 두 개 있습니다.

| 항목 | CIDR | 용도 |
|---|---|---|
| Pod Network | `10.128.0.0/14` | Pod 간 통신 (OVN-Kubernetes geneve 터널) |
| Kubernetes Service CIDR | `172.30.0.0/16` | Service ClusterIP 가상 IP |

> **이름 충돌 주의**
> 우리가 정의한 "Service Network" (192.168.20.0/24)와 Kubernetes의 "Service CIDR" (172.30.0.0/16)은 이름이 같지만 완전히 다른 개념입니다.
> - **External Service Network (192.168.20.0/24)**: 우리 PoC에서 정의한 외부 서비스 트래픽용 물리/논리 네트워크
> - **Kubernetes Service CIDR (172.30.0.0/16)**: Kubernetes Service 리소스에 할당되는 가상 ClusterIP 대역
>
> 문서 전반에서 혼동을 피하기 위해 **External Service Network**와 **Kubernetes Service CIDR**로 구분해 표기합니다.

---

## 1.4 machineNetwork의 의미 (심화)

`machineNetwork`는 install-config.yaml에서 가장 중요하면서도 가장 오해받기 쉬운 항목입니다. 이 항목의 의미를 정확히 이해해야 이후 모든 네트워크 설계가 일관됩니다.

### 1.4.1 정의

`machineNetwork`는 **OpenShift 클러스터를 구성하는 노드(머신)들이 위치하는 IP 대역**을 선언하는 항목입니다. 즉, master/worker 노드의 호스트 IP가 속한 네트워크 CIDR입니다.

```yaml
networking:
  machineNetwork:
  - cidr: 192.168.10.0/24
```

이 선언은 "이 클러스터의 모든 노드는 192.168.10.0/24 대역의 IP를 가진다"는 의미입니다.

### 1.4.2 machineNetwork가 결정하는 것들

이 한 줄의 설정이 다음 다섯 가지를 모두 결정합니다.

**1. 노드 InternalIP 선정 기준**

`kubelet`과 `nodeip-configuration` 서비스는 machineNetwork CIDR과 일치하는 NIC의 IP를 노드의 InternalIP로 자동 선택합니다. 노드에 NIC가 여러 개라면 machineNetwork에 속한 IP가 있는 NIC가 primary로 선택됩니다.

본 설계에서 podpool1은 bond0(192.168.10.61)과 bond1(192.168.20.61)을 모두 가지지만, machineNetwork에 192.168.10.0/24만 명시했기 때문에 bond0의 IP가 InternalIP로 잡힙니다.

**2. API 서버와 etcd 통신 경로**

Master 간 etcd peer 통신, API 서버 endpoint, controller-manager의 leader election이 모두 InternalIP를 사용합니다. machineNetwork는 노드 간 안정적이고 저지연인 망이어야 합니다.

**3. Machine Config Server (MCS) 접근 경로**

Bootstrap 단계에서 master/worker는 22623 포트로 ignition을 가져오는데, 이 트래픽도 machineNetwork 안에서 일어납니다. LB의 22623 backend는 machineNetwork에 있는 master IP를 바라봐야 정상 동작합니다.

**4. OVN-Kubernetes 노드 간 터널 종단점**

OVN-Kubernetes는 노드 간 Pod 트래픽을 Geneve(UDP 6081) 터널로 캡슐화하는데, 터널의 양쪽 종단점이 노드의 InternalIP입니다. 따라서 machineNetwork는 모든 노드 간 UDP 6081이 통하는 망이어야 합니다.

**5. CSR 자동 승인 검증**

`kubelet`이 부팅 시 자체 CSR을 발급할 때 hostname과 InternalIP를 SAN에 포함합니다. `cluster-machine-approver`는 이 SAN이 machineNetwork CIDR에 속하는지 검증해 자동 승인 여부를 결정합니다. machineNetwork와 실제 노드 IP가 어긋나면 CSR 자동 승인이 거부됩니다.

### 1.4.3 단일 CIDR vs 다중 CIDR

`machineNetwork`는 배열이라 여러 CIDR을 넣을 수 있지만, 다음 경우에만 의미가 있습니다.

- IPv4/IPv6 dual stack 구성
- master와 worker가 서로 다른 L2 segment에 있고 라우팅으로 연결된 경우
- 노드 추가 시 기존 CIDR을 확장할 수 없어 별도 대역을 추가하는 경우

본 PoC처럼 **모든 노드가 같은 Infra망(192.168.10.0/24)에 있다면 단일 CIDR이 정답**입니다.

> **Service망을 machineNetwork에 추가하면 안 되는 이유**
> Service망(192.168.20.0/24)을 machineNetwork에 추가하면 "이 망에도 노드 IP가 있을 수 있다"는 선언이 됩니다. 그러면 OpenShift가 Service망 IP를 노드 통신 경로로 선택할 가능성이 생깁니다.
>
> 결과:
> - 노드 InternalIP가 의도와 다른 NIC로 잡힐 수 있음
> - OVN geneve 터널이 Service망을 경유할 수 있음
> - CSR 자동 승인 기준이 흔들림
> - 운영 트러블슈팅이 광범위하게 어려워짐
>
> **Service망은 OpenShift 입장에서 "그냥 노드에 추가로 붙어있는 IP"로 두는 것이 핵심입니다.** OpenShift 제어 평면 바깥에 두고, 노드 OS와 워크로드 레벨에서만 사용합니다.

### 1.4.4 검증 명령어

설치 후 machineNetwork 선언과 실제 노드 상태가 일치하는지 검증합니다.

```bash
# 모든 노드의 InternalIP가 192.168.10.x인지 확인
oc get nodes -o wide
```

기대 출력:
```
NAME                STATUS   ROLES         INTERNAL-IP      ...
master-0            Ready    master        192.168.10.31    ...
master-1            Ready    master        192.168.10.32    ...
master-2            Ready    master        192.168.10.33    ...
ap-worker-0         Ready    ap            192.168.10.41    ...
ap-worker-1         Ready    ap            192.168.10.42    ...
db-worker-0         Ready    db            192.168.10.51    ...
db-worker-1         Ready    db            192.168.10.52    ...
podpool1-worker-0   Ready    podpool1      192.168.10.61    ...  ← 192.168.20.61이 아님
podpool1-worker-1   Ready    podpool1      192.168.10.62    ...
```

```bash
# 특정 노드의 InternalIP를 명시적으로 확인
oc get node podpool1-worker-0 -o jsonpath='{.status.addresses}' | jq
```

기대 출력:
```json
[
  {"address": "192.168.10.61", "type": "InternalIP"},
  {"address": "podpool1-worker-0.ocp1.example.com", "type": "Hostname"}
]
```

노드 OS 내부에서 확인:
```bash
# kubelet의 노드 IP 설정 확인
cat /etc/systemd/system/kubelet.service.d/20-nodenet.conf
# 기대: KUBELET_NODE_IP=192.168.10.61

# default route 확인
ip route show default
# 기대: default via 192.168.10.1 dev bond0

# 주요 포트의 listen IP 확인
ss -tlnp | grep -E ':(6443|22623|10250)'
```

---

## 1.5 IP 대역 설계

### 1.5.1 Infra + NAS Network (192.168.10.0/24)

| 용도 | IP |
|---|---|
| Gateway | 192.168.10.1 |
| Bastion (DNS, Mirror Registry, HAProxy) | 192.168.10.10 |
| API VIP / MCS VIP | 192.168.10.20 |
| Default Ingress VIP (관리용) | 192.168.10.21 |
| Bootstrap (설치 후 제거) | 192.168.10.30 |
| Master 0 | 192.168.10.31 |
| Master 1 | 192.168.10.32 |
| Master 2 | 192.168.10.33 |
| AP Worker 0 | 192.168.10.41 |
| AP Worker 1 | 192.168.10.42 |
| DB Worker 0 | 192.168.10.51 |
| DB Worker 1 | 192.168.10.52 |
| PodPool1 Worker 0 (Infra IP) | 192.168.10.61 |
| PodPool1 Worker 1 (Infra IP) | 192.168.10.62 |
| NAS VIP / NFS VIP | 192.168.10.70 |
| NAS Controller 1 | 192.168.10.71 |
| NAS Controller 2 | 192.168.10.72 |

### 1.5.2 Service Network (192.168.20.0/24)

| 용도 | IP |
|---|---|
| Gateway | 192.168.20.1 |
| Bastion (Service NIC, HAProxy) | 192.168.20.10 |
| Service Ingress VIP (업무용) | 192.168.20.20 |
| Container Egress IP Pool | 192.168.20.30 ~ 192.168.20.49 |
| PodPool1 Worker 0 (Service IP) | 192.168.20.61 |
| PodPool1 Worker 1 (Service IP) | 192.168.20.62 |
| VM Service IP Pool (VM Guest용) | 192.168.20.100 ~ 192.168.20.199 |
| Reserved | 192.168.20.200 ~ 192.168.20.249 |

### 1.5.3 IP 설계의 학습 포인트

**1. VIP는 노드 IP와 분리**

API VIP(.20), Default Ingress VIP(.21), Service Ingress VIP(20.20)는 모두 노드 IP와 별도로 할당했습니다. PoC에서는 Bastion 단일 노드가 처리하지만, 향후 LB가 이중화되거나 다른 장비로 이전될 때 VIP만 따라 움직이면 됩니다.

**2. ap/db는 .40/.50대, podpool1은 .60대로 그룹화**

역할별로 IP 대역을 시각적으로 구분하면 운영자가 IP만 봐도 노드 역할을 추정할 수 있습니다. 운영 가이드와 트러블슈팅 문서에서 IP 인용 시 가독성이 좋아집니다.

**3. PodPool1만 두 대역에서 IP 사용**

PodPool1만 Infra(.10.61~)와 Service(.20.61~) 두 대역에서 같은 마지막 옥텟을 사용합니다. 노드별로 두 IP가 동일한 마지막 숫자라 라우팅 추적이 쉽습니다.

**4. VM Service IP Pool은 별도 대역**

VM이 사용할 IP(.20.100~199)는 노드 IP(.20.61~62) 및 LB VIP(.20.20)와 충분히 떨어진 대역에 둡니다. DHCP 또는 NetworkAttachmentDefinition 설정 시 충돌 방지.

---

## 1.6 노드별 NIC / Bond 설계

각 노드의 NIC 구성은 노드 역할과 직결됩니다.

### 1.6.1 Master 노드

```
bond0: Infra + NAS Network
  - IP: 192.168.10.31 / 32 / 33
  - default gateway: 192.168.10.1
  - 용도: API, DNS, MCS, Mirror Registry, NAS, etcd peer
```

**Master는 Service망에 연결하지 않습니다.** Master는 클러스터 제어 평면 전용으로 유지하고, 업무 서비스 트래픽은 PodPool1 또는 VM 노드에서 처리합니다.

> **왜 Master에 Service망 NIC를 두지 않는가**
> - Master는 control plane 부하만 처리하도록 격리
> - Service망 트래픽이 master에 도달하지 못하게 함으로써 보안 경계 강화
> - 관리 사이클에서 master는 inbound 진입점이 아니므로 Service NIC 불필요
> - NIC 수를 줄여 H/W 자원 효율화

### 1.6.2 VM 전용 ap/db 노드

```
bond0: Infra + NAS Network
  - IP: 192.168.10.41~42 / 192.168.10.51~52
  - default gateway: 192.168.10.1
  - 용도: API, DNS, MCS, Mirror, NAS, OVN node primary IP

bond1: VM Service Network
  - IP 없음 (호스트는 사용하지 않음)
  - Linux bridge "br-vm-svc" 생성
  - NetworkAttachmentDefinition을 통해 VM Guest OS에 노출
  - VM Guest는 192.168.20.x 사용
```

구조 도식:
```
bond0 (Infra)
  └─ RHCOS host IP (192.168.10.41)
  └─ default gateway
  └─ kubelet / API / DNS / Mirror / NAS

bond1 (Service)
  └─ br-vm-svc (Linux bridge, IP 없음)
       └─ NetworkAttachmentDefinition: vm-service-net
            └─ VM NIC (Guest OS가 192.168.20.x 사용)
```

> **핵심 개념: VM 노드의 Service망은 "노드가 사용하는 망"이 아니라 "VM에게 제공하는 망"**
> 호스트 OS(RHCOS)는 bond1에 IP를 가지지 않습니다. bond1은 단순히 VM Guest OS가 외부 Service망과 통신할 수 있도록 L2 통로 역할만 합니다.
> 이 구성으로 VM은 마치 물리 네트워크에 직접 연결된 것처럼 동작하며, 호스트 OS는 VM 서비스 트래픽에 관여하지 않습니다.
>
> **외부 → VM 트래픽 흐름 (Bastion 미경유):**
> ```
> 외부 사용자 → Service망 스위치 (192.168.20.0/24)
>             → ap/db 호스트의 ens224 (NIC, IP 없음)
>             → br-vm-svc (Linux bridge, IP 없음)
>             → NetworkAttachmentDefinition: vm-service-net
>             → VM Guest OS (eth1: 192.168.20.150)
> ```
> 이 경로의 어느 단계에도 Bastion HAProxy가 등장하지 않습니다. Service망 스위치만 통과하면 VM에 직접 도달합니다.

### 1.6.3 Container 전용 podpool1 노드

```
bond0: Infra + NAS Network
  - IP: 192.168.10.61 / 192.168.10.62
  - default gateway: 192.168.10.1
  - 용도: API, DNS, MCS, Mirror, NAS, OVN node primary IP

bond1: Container Service Network
  - IP: 192.168.20.61 / 192.168.20.62
  - 용도: Service IngressController (HostNetwork), 컨테이너 업무 Egress
  - default gateway 없음 (Infra가 default)
```

구조 도식:
```
bond0 (Infra)
  └─ RHCOS host IP (192.168.10.61)
  └─ default gateway (192.168.10.1)
  └─ kubelet / API / DNS / Mirror / NAS / MTV StorageClass

bond1 (Service)
  └─ RHCOS host IP (192.168.20.61)
  └─ Service IngressController가 0.0.0.0:443에서 listen
  └─ 컨테이너 업무 Egress 출구
```

> **PodPool1만 두 NIC에 IP를 갖는 이유**
> - bond0(Infra)는 OpenShift 노드 자체 통신 (kubelet → API 등)
> - bond1(Service)은 외부 사용자 트래픽의 진입점이자 업무 컨테이너의 출구
> - 두 망 모두에서 호스트가 능동적으로 트래픽을 주고받아야 하므로 두 NIC 모두 IP 필요
> - 단, **default gateway는 Infra에만 두어 관리 통신 안정성 확보**

### 1.6.4 NIC 구성 비교표

| 노드 역할 | bond0 (Infra) | bond1 (Service) | 비고 |
|---|---|---|---|
| Master | IP O, GW O | (없음) | control plane 전용 |
| AP/DB Worker | IP O, GW O | IP X, bridge만 | VM에 Service망 제공 |
| PodPool1 Worker | IP O, GW O | IP O, GW X | 호스트가 두 망 모두 사용 |
| Bastion | IP O (.10.10), GW O | IP O (.20.10) | LB가 두 망 처리 |

---

## 1.7 Default Gateway 정책

### 1.7.1 기본 정책

**모든 노드의 default gateway는 Infra망(192.168.10.1)에 둡니다.**

대상:
- Master → 192.168.10.1
- AP/DB Worker → 192.168.10.1
- PodPool1 Worker → 192.168.10.1
- Bastion → 192.168.10.1

### 1.7.2 이유

1. **API/MCS/DNS/Mirror/NAS 접근 안정성** — 노드가 모르는 IP로 패킷을 보낼 때 가장 자주 가는 곳이 이 다섯 서비스이고, 모두 Infra망에 있음
2. **MachineConfig 적용 중 장애 위험 감소** — 노드 재구성 중에도 Infra 경로가 살아있어야 MCO가 작업 완료 가능
3. **노드 Ready 상태 안정성** — kubelet → API 통신이 default gateway에 의존하지 않더라도, 기타 시스템 트래픽이 default route를 따르므로 Infra 안정성이 노드 상태에 직결
4. **운영 장애 분석 단순화** — 모든 노드의 default route가 동일하므로 라우팅 문제 분석이 일관됨

### 1.7.3 Service망으로 가야 하는 트래픽 처리

업무 트래픽이 Service망으로 흘러야 하는 경우는 default gateway가 아닌 별도 메커니즘으로 처리합니다.

| 트래픽 | 처리 방법 |
|---|---|
| 업무 Pod의 외부 호출 (egress) | OVN-Kubernetes EgressIP (PoC 검증) 또는 정책 라우팅 |
| Service IngressController inbound | LB → podpool1 Service NIC (connection tracking으로 응답 반환) |
| VM Guest OS의 외부 통신 | VM Guest OS의 자체 default gateway (192.168.20.1) |
| Service망 VIP 라우팅 | 외부 LB가 직접 처리 |

> **흔한 실수: 노드에 두 개의 default route 설정**
> Service망에도 default gateway가 있다고 가정하고 노드에 두 개의 default route를 설정하면, 라우팅 우선순위에 따라 트래픽이 예측 불가능하게 흐릅니다. NetworkManager가 metric으로 우선순위를 정하지만, 노드 재부팅이나 NIC 순서 변경 시 동작이 바뀔 수 있습니다.
>
> **default route는 반드시 Infra 하나만 두십시오.** Service망으로 강제할 outbound는 정책 라우팅(ip rule) 또는 EgressIP로 명시적으로 처리합니다.

---

## 1.8 Ingress 설계

### 1.8.1 두 IngressController 분리 원칙

OpenShift는 IngressController를 여러 개 운영할 수 있습니다. 본 PoC는 **관리용**과 **업무용**을 분리합니다.

| 구분 | Default IngressController | Service IngressController |
|---|---|---|
| 용도 | OpenShift Console, OAuth, 관리 Route | 사용자 **컨테이너** 업무 애플리케이션 Route |
| 배치 노드 | ap MCP (PoC) / infra MCP (프로덕션) | podpool1 MCP |
| 배포 방식 | HostNetwork, 80/443 | HostNetwork, 80/443 |
| 도메인 | `apps.<cluster>.<domain>` | `svcapps.<cluster>.<domain>` |
| VIP | 192.168.10.21 (Bastion HAProxy Infra 측) | 192.168.20.20 (Bastion HAProxy Service 측) |
| 대상 망 | Infra Network | Service Network |
| 적용 대상 | OpenShift 자동 Route + 관리 Route | **컨테이너 업무 Route만** (VM 서비스는 별개) |

> **VIP가 적용되는 트래픽 범위**
> 위 표의 VIP는 모두 **OpenShift Route 객체를 통한 트래픽**에 적용됩니다.
> - VM은 OpenShift Route를 사용하지 않으므로 이 VIP를 거치지 않습니다.
> - VM은 Service망 L2 직결로 외부와 통신 (Part 1.6.2 참조).
> - 즉, Service Ingress VIP(20.20)는 **컨테이너 업무 전용 진입점**입니다.

### 1.8.2 도메인 분리의 의미

OpenShift에서 Route를 만들면 host가 `<route-name>-<namespace>.<ingresscontroller-domain>` 형식으로 자동 생성됩니다. IngressController의 wildcard 도메인이 분리되어 있으면, **DNS 단계에서 이미 관리 트래픽과 서비스 트래픽이 갈라집니다.**

자동 생성되는 관리 Route 예시 (default IngressController):

| Namespace | Route 이름 | 자동 생성 host |
|---|---|---|
| openshift-console | console | `console-openshift-console.apps.ocp1.example.com` |
| openshift-authentication | oauth-openshift | `oauth-openshift.apps.ocp1.example.com` |
| openshift-monitoring | thanos-querier | `thanos-querier-openshift-monitoring.apps.ocp1.example.com` |
| openshift-monitoring | alertmanager-main | `alertmanager-main-openshift-monitoring.apps.ocp1.example.com` |

사용자가 만드는 업무 Route 예시 (service IngressController):
```
frontend.svcapps.ocp1.example.com
api.svcapps.ocp1.example.com
```

DNS 등록:
```
*.apps.ocp1.example.com      IN A   192.168.10.21
*.svcapps.ocp1.example.com   IN A   192.168.20.20
```

이 한 줄씩의 DNS 등록으로 모든 관리 Route는 Infra LB로, 모든 업무 Route는 Service LB로 자동 흐릅니다.

> **교육 포인트**
> 운영자는 새 업무 앱을 배포할 때 host를 명시적으로 지정하지 않아도 됩니다. IngressController의 `routeSelector` 또는 `namespaceSelector`로 자동 매칭되도록 구성하면, namespace에 라벨만 붙이면 Service IngressController가 admit합니다. 자세한 설정은 Part 5에서 다룹니다.

### 1.8.3 Default IngressController 배치 결정 (PoC vs 프로덕션)

> **PoC 환경: ap 노드에 배치**
> ```
> nodeSelector: node-role.kubernetes.io/ap=""
> tolerations: workload=vm:NoSchedule
> ```
> - 장점: 추가 노드 불필요, podpool1과 노드 분리로 80/443 포트 충돌 없음
> - 단점: VM 격리 원칙 일부 양보 (router Pod가 VM 노드에 공존)
> - PoC에서는 자원 효율성과 운영 단순성을 우선

> **프로덕션 환경: infra MCP에 배치**
> - infra 노드 3대 별도 구성
> - infra 노드는 OpenShift 구독에서 면제 (worker 라벨 제거, infra 라벨 부여, 사용자 워크로드 미실행 조건 충족 시)
> - Monitoring, Logging, Image Registry도 함께 infra MCP로 이동
> - 장점: 격리 원칙 완전 유지, HA 향상, 구독 비용 효율
> - PoC → 프로덕션 전환 시 nodeSelector만 `infra`로 변경하고 외부 LB backend 교체

### 1.8.4 Service IngressController 배치

PoC와 프로덕션 모두 **podpool1 MCP**에 배치합니다.

```yaml
nodeSelector: node-role.kubernetes.io/podpool1=""
endpointPublishingStrategy: HostNetwork
domain: svcapps.<cluster>.<domain>
```

> **왜 Service IngressController는 podpool1에 두는가**
> - 업무 컨테이너 Pod가 모이는 노드와 router가 같은 곳에 있으면 노드 내부 통신으로 latency 감소
> - "컨테이너 업무 = podpool1"이라는 단일 규칙으로 운영자 인지 부담 감소
> - Service망 NIC가 podpool1에만 있으므로 router도 자연스럽게 같은 노드에 위치

---

## 1.9 Egress 설계

### 1.9.1 두 종류의 Egress

| Egress 종류 | 트래픽 | 망 | 메커니즘 |
|---|---|---|---|
| 관리 Egress | 노드 자체의 외부 통신 (이미지 pull, NTP, DNS, NAS) | Infra | default gateway (자동) |
| 업무 Egress | 업무 Pod의 외부 시스템 호출 | Service | EgressIP 또는 정책 라우팅 (별도 구성) |

### 1.9.2 관리 Egress (자동)

모든 노드의 default gateway가 Infra(192.168.10.1)이므로, 노드가 명시되지 않은 IP로 보내는 모든 트래픽은 자동으로 Infra망으로 나갑니다. 별도 설정이 필요하지 않습니다.

대표 흐름:
- 노드 → Mirror Registry(192.168.10.10) → 이미지 pull
- 노드 → NTP Server(192.168.10.10 또는 외부 NTP) → 시간 동기화
- 노드 → DNS(192.168.10.10) → 이름 조회
- Pod → NAS VIP(192.168.10.70) → PVC mount

### 1.9.3 업무 Egress (PoC 검증 항목)

업무 Pod가 외부 시스템(예: 외부 DB, 외부 API)을 호출할 때 Service망으로 강제하는 방법은 세 가지입니다.

**옵션 1: OVN-Kubernetes EgressIP (1차 후보)**

```yaml
apiVersion: k8s.ovn.org/v1
kind: EgressIP
metadata:
  name: egress-myapp
spec:
  egressIPs:
  - 192.168.20.30
  namespaceSelector:
    matchLabels:
      egress-network: service
```

- 장점: OpenShift 네이티브 메커니즘, namespace 단위로 깔끔하게 격리
- 제약: secondary NIC(bond1)에 EgressIP를 안정적으로 할당할 수 있는지는 PoC로 검증 필요
- 노드에 `k8s.ovn.org/egress-assignable=""` 라벨 부여 필요

**옵션 2: 정책 라우팅 (대안)**

MachineConfig로 노드 OS에 `ip rule`을 적용해 특정 source IP를 Service NIC로 라우팅.
- 장점: OVN과 무관하게 OS 레벨에서 동작
- 단점: MachineConfig 관리 복잡, Pod IP 변동에 따른 추적 어려움

**옵션 3: 외부 방화벽/NAT (보조)**

업무 Pod가 외부로 나갈 때 Infra 방화벽에서 차단하고, Service망 방화벽에서만 허용. 정책으로 강제.
- 장점: OpenShift 외부에서 통제, 침해 시에도 정책 유효
- 단점: 방화벽 운영 부담

> **PoC 권장: 옵션 1을 시도하고 결과에 따라 옵션 2를 fallback으로 준비**
> EgressIP의 secondary NIC 할당은 OpenShift 버전별 동작이 다를 수 있으므로, PoC 단계에서 실제 동작 검증이 매우 중요합니다. 검증 결과를 문서화하여 프로덕션 의사결정 근거로 활용합니다.

---

## 1.10 PoC vs 프로덕션 차이 요약

본 문서 전체에서 반복되는 PoC와 프로덕션의 차이를 한 표로 정리합니다.

| 항목 | PoC | 프로덕션 권장 |
|---|---|---|
| infra MCP | 없음 (ap에 default Ingress) | 신설 (전용 노드 3대) |
| Default IngressController 배치 | ap MCP | infra MCP |
| Service IngressController 배치 | podpool1 MCP | podpool1 MCP (동일) |
| Monitoring/Logging/Registry | 기본 위치 | infra MCP로 이동 |
| Default Ingress LB | Bastion HAProxy | 관리망 LB 또는 Bastion 이중화 |
| Service Ingress LB | Bastion HAProxy (별도 frontend) | 별도 물리 L4/L7 |
| Bastion NIC | Infra + Service | Infra만 |
| NAS Network | Infra와 통합 | 별도 망 분리 검토 |
| LiveMigration Network | Infra와 공유 | 별도 망 분리 검토 |
| Monitoring Storage | emptyDir 또는 NFS | block storage 권장 |
| LB HA | 단일 Bastion | LB 장비 이중화 |

> **전환 시 변경 영향이 큰 항목**
> 1. **infra MCP 신설** — 노드 추가, 라벨/taint, 인프라 워크로드 일괄 이동
> 2. **Bastion 분리** — Service NIC 제거, Service LB는 별도 장비
> 3. **NAS 망 분리** — NAS 전용 NIC 추가, 라우팅 변경
>
> 이 세 가지를 제외하면 PoC 결과의 대부분이 프로덕션에 그대로 적용 가능합니다.

---

## 1.11 다음 단계

Part 1에서 다룬 설계 결정을 바탕으로 다음 파트들이 진행됩니다.

- **Part 2 (Bastion 및 Load Balancer)**: DNS 레코드, HAProxy 설정, Mirror Registry, NTP/Chrony 위계
- **Part 3 (미러링 및 설치)**: oc-mirror v2, install-config.yaml, RHCOS 설치, 설치 검증
- **Part 4 (Day-2 MCP 및 MachineConfig)**: MCP 분리, MachineConfig 적용, 인증서 관리, etcd 백업
- **Part 5 (Operator 및 Virtualization)**: Operator 설치 순서, OpenShift Virtualization, NMState/NAD, NAS StorageClass
- **Part 6 (시나리오 테스트)**: MTV 마이그레이션, FAR/vBMC fencing, Tekton/GitOps/JBoss, 트러블슈팅

---

## 1.12 Part 1 학습 점검

다음 질문에 답할 수 있다면 Part 1을 충분히 학습한 것입니다.

1. 본 PoC가 두 개의 외부 네트워크를 분리한 이유는 무엇인가?
2. `machineNetwork`에 Service망(192.168.20.0/24)을 추가하면 어떤 문제가 발생하는가?
3. PodPool1 노드는 왜 두 NIC에 모두 IP를 가지는데 ap/db 노드의 bond1에는 IP가 없는가?
4. 모든 노드의 default gateway를 Infra망에 두는 이유 세 가지를 설명하시오.
5. Default IngressController와 Service IngressController를 분리하는 이유와, 각각의 도메인이 분리되어야 하는 이유는 무엇인가?
6. PoC에서 default IngressController를 ap에 두는 결정의 trade-off는 무엇인가?
7. 업무 Pod의 outbound를 Service망으로 강제하는 세 가지 방법은?
8. PoC에서 프로덕션으로 전환할 때 가장 큰 변경 세 가지는 무엇인가?

---

*Part 1 끝. 다음은 Part 2 (Bastion 및 Load Balancer) 입니다.*
