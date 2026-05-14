# Part 1. 아키텍처 및 네트워크 설계

> **문서 성격**  
> 본 문서는 폐쇄망 환경에서 OpenShift Container Platform 4.20을 UPI 방식으로 설치하고, VM 워크로드와 컨테이너 워크로드를 분리 운영하며, MTV(Migration Toolkit for Virtualization) 및 FAR(Fence Agents Remediation) 시나리오를 검증하기 위한 **교육자료 + PoC 설계 가이드**입니다.
>
> 단순한 절차 모음이 아니라 **“왜 이렇게 설계하는가”** 를 학습할 수 있도록 각 결정의 배경, 대안, 흔한 실수, 검증 포인트를 함께 다룹니다.

---

## 1.1 문서 범위와 학습 목표

### 1.1.1 다루는 환경

| 항목 | 내용 |
|---|---|
| 제품 | OpenShift Container Platform 4.20 |
| 설치 방식 | UPI, User Provisioned Infrastructure |
| 네트워크 플러그인 | OVN-Kubernetes |
| 환경 | 폐쇄망, Disconnected |
| Bastion 역할 | DNS, Load Balancer, Mirror Registry, Ignition 제공, vBMC, NTP |
| 미러링 방식 | oc-mirror plugin v2 |
| 가상화 | OpenShift Virtualization |
| 마이그레이션 | MTV, Migration Toolkit for Virtualization |
| 스토리지 | PoC 기준 NAS StorageClass |
| Fencing 테스트 | FAR + vBMC |
| 워크로드 분리 | VM 전용 노드풀 / 컨테이너 전용 노드풀 |

---

### 1.1.2 기준 도메인

| 항목 | 값 |
|---|---|
| Cluster Name | `ocp420` |
| Base Domain | `aonsoft.demo.com` |
| Cluster FQDN | `ocp420.aonsoft.demo.com` |
| Default Apps Domain | `apps.ocp420.aonsoft.demo.com` |
| Service Apps Domain | `svcapps.ocp420.aonsoft.demo.com` |

주요 DNS 이름은 다음과 같이 구성합니다.

| 용도 | FQDN | VIP |
|---|---|---|
| Kubernetes API | `api.ocp420.aonsoft.demo.com` | `192.168.10.20` |
| Internal API | `api-int.ocp420.aonsoft.demo.com` | `192.168.10.20` |
| Default Apps | `*.apps.ocp420.aonsoft.demo.com` | `192.168.10.21` |
| Service Apps | `*.svcapps.ocp420.aonsoft.demo.com` | `192.168.20.20` |

---

### 1.1.3 핵심 설계 원칙

| 원칙 | 설명 |
|---|---|
| Infra / Service 분리 | 관리 트래픽은 Infra망, 업무 서비스 트래픽은 Service망 사용 |
| Master 전용성 유지 | Master는 control plane 전용으로 사용하고 Service망에 연결하지 않음 |
| VM / Container 분리 | VM 워크로드와 컨테이너 워크로드를 다른 MCP에서 운영 |
| PoC / 프로덕션 차이 명시 | PoC의 절충안과 운영 전환 시 변경점을 문서에 계속 표시 |
| NAS는 PoC에서 Infra 통합 | PoC에서는 NAS를 Infra와 통합하되, 운영에서는 분리 검토 |
| node primary IP 고정 | 모든 노드의 InternalIP는 Infra망 IP가 되도록 설계 |

---

### 1.1.4 학습 목표

본 문서를 학습한 후 학습자는 다음을 설명하고 수행할 수 있어야 합니다.

- OpenShift UPI 설치를 위한 네트워크 구조를 직접 설계할 수 있다.
- `machineNetwork`, `clusterNetwork`, `serviceNetwork`의 차이를 설명할 수 있다.
- Default IngressController와 Service IngressController를 분리하는 이유를 설명할 수 있다.
- 노드별 NIC/Bond 구성을 트래픽 흐름과 연결해서 이해할 수 있다.
- podpool1 노드가 Infra IP와 Service IP를 모두 가지는 이유를 설명할 수 있다.
- PoC 설계와 프로덕션 설계의 차이를 구분할 수 있다.
- 네트워크 문제 발생 시 어떤 지점부터 확인해야 하는지 판단할 수 있다.

---

## 1.2 전체 아키텍처

### 1.2.1 노드 구성

| 역할 | 수량 | MCP | 주요 용도 |
|---|---:|---|---|
| Bastion | 1 | 해당 없음 | DNS, LB, Mirror Registry, Ignition 제공, vBMC, NTP |
| Master | 3 | `master` | API Server, etcd, scheduler, controller |
| VM Worker AP | 2 | `ap` | AP성 VM, Default IngressController 예외 배치 |
| VM Worker DB | 2 | `db` | DB성 VM |
| Container Worker | 2 | `podpool1` | 컨테이너 업무 워크로드, Service IngressController |

---

### 1.2.2 ap와 db를 분리하는 이유

`ap`와 `db`는 모두 VM 전용 Worker이며 NIC 구성도 동일합니다.  
그럼에도 별도 MCP로 분리하는 이유는 운영 정책 분리를 검증하기 위해서입니다.

분리 근거는 다음과 같습니다.

- AP성 VM과 DB성 VM의 장애 영향 범위 분리
- 향후 CPU pinning / hugepages / reserved CPU 정책 차등 적용 가능
- kdump / crashkernel 설정 차등 적용 가능
- multipath 또는 스토리지 정책 차등 적용 가능
- AP성 VM과 DB성 VM의 운영 패치 윈도우 분리
- VM migration 정책 또는 nodeSelector 정책 분리 가능

> **운영 단순성 우선 시 대안**  
> 운영 단순성이 더 중요하다면 `ap`와 `db`를 단일 `vm-worker` MCP로 통합할 수 있습니다. 본 PoC는 운영 정책 분리 검증을 포함하므로 `ap`, `db`를 분리합니다.

---

### 1.2.3 ap 노드의 예외 역할

본 PoC에서 `ap` 노드는 VM 실행 노드를 기본으로 합니다.  
다만 infra 전용 Worker가 없기 때문에 **관리용 Default IngressController 배치를 예외적으로 허용**합니다.

```text
ap 노드 = VM 중심 노드 + 관리 Ingress 예외 허용 노드
```

이는 PoC 환경에서 노드 수를 줄이기 위한 절충안입니다.  
프로덕션에서는 Default IngressController를 `ap`가 아니라 별도 `infra` MCP로 이동하는 것을 권장합니다.

---

### 1.2.4 전체 구성도

```text
                          [ 외부 관리자 ]      [ 외부 사용자 ]
                                 |                  |
                                 v                  v
                    +--------------------+   +--------------------+
                    | Bastion HAProxy    |   | Bastion HAProxy    |
                    | Infra VIP          |   | Service VIP        |
                    | 192.168.10.20/21   |   | 192.168.20.20      |
                    +--------------------+   +--------------------+
                                 |                  |
                  +--------------+--------+         |
                  |              |        |         |
                  v              v        v         v
              [Master x3]    [AP x2]   [DB x2]   [PodPool1 x2]
              Infra NIC      Infra +   Infra +   Infra + Service
                             VM-svc    VM-svc    NIC
                             bridge    bridge

   Infra + NAS Network (192.168.10.0/24) ─────────────────────┐
                                                              |
   Service Network    (192.168.20.0/24) ────────────────┐     |
                                                        |     |
                                                        v     v
                                                  [Service Router]
                                                  [VM Service NIC]
```

---

### 1.2.5 트래픽 흐름 요약

| 트래픽 종류 | 방향 | 사용 망 | 진입/출구 |
|---|---|---|---|
| API / kubelet / MCS | 노드 ↔ API | Infra | API VIP `192.168.10.20` |
| etcd peer | Master ↔ Master | Infra | Master InternalIP |
| 이미지 pull | 노드 → Mirror | Infra | Mirror Registry `192.168.10.10` |
| DNS | 노드 → DNS | Infra | Bastion DNS |
| NTP / Chrony | 노드 → NTP | Infra | Bastion 또는 상위 NTP |
| Console / OAuth | 외부 → 내부 | Infra | Default IngressController, ap 노드 |
| 업무 앱 inbound | 외부 → 내부 | Service | Service IngressController, podpool1 노드 |
| 업무 앱 outbound | 내부 → 외부 | Service | EgressIP 또는 정책 라우팅, PoC 검증 항목 |
| VM inbound | 외부 → VM | Service | VM Service NAD |
| VM outbound | VM → 외부 | Service | VM Guest OS 라우팅 |
| NAS storage | 노드 ↔ NAS | Infra, PoC | NAS VIP `192.168.10.70` |
| Monitoring / Logging 내부 통신 | 내부 ↔ 내부 | Pod Network / Infra | 기본 모니터링 스택 |

> **교육 포인트**  
> 운영자가 “이 트래픽은 어느 망으로 가야 하지?”를 매번 고민하지 않도록, **트래픽의 성격이 결정되면 사용하는 망도 자동으로 결정**되도록 설계하는 것이 핵심입니다.

---

## 1.3 네트워크 설계

### 1.3.1 네트워크 분리 원칙

| 네트워크 | CIDR | 용도 |
|---|---|---|
| Infra + NAS Network | `192.168.10.0/24` | 클러스터 관리 + PoC NAS |
| External Service Network | `192.168.20.0/24` | 업무 서비스 + VM 서비스 |

---

### 1.3.2 Infra + NAS Network

Infra + NAS Network는 다음 트래픽을 처리합니다.

- Kubernetes API Server 통신
- Machine Config Server, MCS 통신
- kubelet ↔ API Server 통신
- etcd peer 통신
- DNS 조회
- Mirror Registry 이미지 pull
- NAS StorageClass 접근, PoC 한정
- NTP / Chrony 동기화
- OpenShift Console / OAuth 접근
- OVN-Kubernetes 노드 간 터널 통신

> **포인트**  
> PoC에서는 NAS를 Infra망과 통합합니다. 운영 환경에서는 NAS 트래픽이 커질 수 있으므로 별도 NAS망 분리를 검토해야 합니다.

---

### 1.3.3 External Service Network

External Service Network는 다음 트래픽을 처리합니다.

- 컨테이너 업무 애플리케이션 Ingress
- 컨테이너 업무 애플리케이션 Egress
- VM 서비스 트래픽
- Service Mesh 외부 노출 트래픽
- 외부 사용자 → 업무 애플리케이션 접근

---

### 1.3.4 NAS를 Infra망에 통합한 이유

운영 환경이라면 NAS는 별도 망으로 분리하는 것이 더 바람직합니다.

```text
Infra Network: 192.168.10.0/24
Service Network: 192.168.20.0/24
NAS Network: 192.168.30.0/24
```

하지만 본 PoC에서는 자원과 구성 단순성을 고려해 NAS를 Infra망과 통합합니다.

```text
PoC 기준:
Infra + NAS Network: 192.168.10.0/24
```

운영 전환 시 검토 항목은 다음과 같습니다.

- NAS 전용 NIC
- NAS 전용 VLAN
- NAS 전용 StorageClass
- LiveMigration 또는 MTV 트래픽 분리

---

## 1.4 관리 사이클과 서비스 사이클

### 1.4.1 관리 사이클

관리 사이클은 Infra망 안에서 닫히도록 설계합니다.

```text
관리자
  → Infra LB
  → Default IngressController, ap 노드 Infra NIC
  → console / oauth Pod
  → 응답
  → Infra 경로
  → 관리자
```

특징:

- Infra VIP 사용
- Default IngressController 사용
- `*.apps.ocp420.aonsoft.demo.com` 도메인 사용
- ap 노드에 배치
- Console / OAuth / 관리 Route 처리

---

### 1.4.2 서비스 사이클

서비스 사이클은 Service망을 통해 들어오도록 설계합니다.

```text
사용자
  → Service LB
  → Service IngressController, podpool1 Service NIC
  → 업무 Pod
  → 응답
  → 동일 연결의 응답으로 처리
  → 사용자
```

구분해야 할 항목은 다음과 같습니다.

| 구분 | 설명 |
|---|---|
| Ingress 응답 | 외부에서 들어온 연결에 대한 응답 |
| 신규 Egress | Pod가 새롭게 외부로 연결을 시작하는 트래픽 |

Service Ingress는 외부 LB가 podpool1의 Service IP로 직접 연결하므로, 해당 연결에 대한 응답은 커널 연결 추적 및 로컬 라우팅에 따라 동일 연결의 응답으로 처리됩니다.

하지만 업무 Pod가 외부 시스템으로 **새로운 outbound 연결**을 만들 때는 기본적으로 노드의 default route 영향을 받을 수 있습니다.  
따라서 업무 Egress를 Service망으로 강제하려면 별도 구성이 필요합니다.

후보:

- OVN-Kubernetes EgressIP
- 정책 라우팅
- 외부 방화벽 / NAT 정책

> **흔한 실수: 비대칭 라우팅**  
> Service망에도 default gateway를 설정하면 kubelet → API Server 통신 같은 관리 트래픽까지 Service망으로 나가려 할 수 있습니다. 그 결과 node InternalIP 선정, CSR 승인, OVN 터널, 노드 Ready 상태가 흔들릴 수 있습니다.
>
> **모든 노드의 default gateway는 반드시 Infra망에 둡니다.**

---

## 1.5 Kubernetes 내부 네트워크

| 항목 | CIDR | 용도 |
|---|---|---|
| Pod Network | `10.128.0.0/14` | Pod 간 통신, OVN-Kubernetes |
| Kubernetes Service CIDR | `172.30.0.0/16` | Service ClusterIP |

### 1.5.1 이름 충돌 주의

| 이름 | CIDR | 의미 |
|---|---|---|
| External Service Network | `192.168.20.0/24` | 외부 업무 서비스 트래픽용 물리/논리 네트워크 |
| Kubernetes Service CIDR | `172.30.0.0/16` | Kubernetes Service 리소스의 ClusterIP 대역 |

> **교육 포인트**  
> “Service Network”라는 표현이 혼동을 만들 수 있으므로, 문서에서는 가능한 한 `External Service Network`와 `Kubernetes Service CIDR`로 구분해 표기합니다.

---

## 1.6 machineNetwork의 의미

`machineNetwork`는 `install-config.yaml`에서 가장 중요하면서도 가장 오해받기 쉬운 항목입니다.

### 1.6.1 정의

`machineNetwork`는 OpenShift가 노드의 primary/InternalIP로 사용할 IP 대역을 선언하는 항목입니다.

```yaml
networking:
  machineNetwork:
  - cidr: 192.168.10.0/24
```

이 선언의 의미는 다음과 같습니다.

```text
OpenShift가 노드의 primary/InternalIP로 사용할 IP 대역은 192.168.10.0/24이다.
```

노드가 추가 NIC를 통해 다른 대역의 IP를 가질 수는 있습니다. 예를 들어 podpool1은 `192.168.20.x` Service IP도 가질 수 있습니다.

하지만 본 설계의 원칙은 다음입니다.

```text
node InternalIP는 반드시 192.168.10.x가 되어야 한다.
192.168.20.x는 node InternalIP로 사용하지 않는다.
```

---

### 1.6.2 machineNetwork가 결정하는 것

| 항목 | 설명 |
|---|---|
| Node InternalIP | kubelet이 노드 IP로 사용할 대역 |
| API / kubelet 통신 | 노드가 API Server와 통신하는 경로 |
| MCS 접근 | bootstrap/master/worker가 22623으로 접근하는 경로 |
| OVN-Kubernetes 터널 | Geneve 터널 종단점으로 사용할 노드 IP |
| CSR / 노드 등록 | 노드 이름, IP, CSR 정보의 일관성 |

---

### 1.6.3 단일 CIDR을 사용하는 이유

본 PoC에서는 모든 노드의 primary/InternalIP를 Infra망에 둡니다.

```yaml
machineNetwork:
- cidr: 192.168.10.0/24
```

Service망을 `machineNetwork`에 추가하지 않습니다.

```yaml
# 사용하지 않음
machineNetwork:
- cidr: 192.168.10.0/24
- cidr: 192.168.20.0/24
```

---

### 1.6.4 Service망을 machineNetwork에 넣으면 안 되는 이유

Service망을 `machineNetwork`에 넣으면 OpenShift는 다음처럼 해석할 수 있습니다.

```text
192.168.20.0/24도 노드 primary IP가 될 수 있는 대역이다.
```

그 결과 다음 문제가 발생할 수 있습니다.

- podpool1의 InternalIP가 `192.168.20.61`로 잡힐 수 있음
- OVN Geneve 터널이 Service망을 경유할 수 있음
- kubelet → API 통신 경로가 흔들릴 수 있음
- CSR 승인 또는 노드 등록 과정에서 문제가 발생할 수 있음
- 네트워크 트러블슈팅 범위가 크게 늘어남

> **핵심 원칙**  
> Service망은 OpenShift 제어 평면 관점에서는 “노드 primary network”가 아닙니다. Service망은 노드 OS와 워크로드 수준에서 사용하는 보조 업무망으로만 둡니다.

---

### 1.6.5 CSR 승인 관련 주의

UPI 환경에서는 CSR 승인이 자동으로 완료되지 않거나, 수동 승인이 필요한 경우가 있습니다.

CSR 승인 과정에서는 다음 정보의 일관성이 중요합니다.

- kubelet이 요청한 노드 이름
- 노드 IP
- CSR의 SAN 정보
- 클러스터가 기대하는 노드 정보
- machineNetwork와 실제 node InternalIP의 일치 여부

설치 검증 단계에서 반드시 다음을 확인합니다.

```bash
oc get csr
oc get nodes -o wide
```

`machineNetwork`와 실제 node InternalIP가 어긋나면 노드 등록, 인증서 승인, 네트워크 플러그인 동작에 문제가 발생할 수 있습니다.

---

### 1.6.6 machineNetwork 검증

설치 후 모든 노드의 InternalIP가 `192.168.10.x`인지 확인합니다.

```bash
oc get nodes -o wide
```

기대 출력 예시:

```text
NAME                STATUS   ROLES         INTERNAL-IP
master-0            Ready    master        192.168.10.31
master-1            Ready    master        192.168.10.32
master-2            Ready    master        192.168.10.33
ap-worker-0         Ready    ap            192.168.10.41
ap-worker-1         Ready    ap            192.168.10.42
db-worker-0         Ready    db            192.168.10.51
db-worker-1         Ready    db            192.168.10.52
podpool1-worker-0   Ready    podpool1      192.168.10.61
podpool1-worker-1   Ready    podpool1      192.168.10.62
```

특정 노드 확인:

```bash
oc get node podpool1-worker-0 -o jsonpath='{.status.addresses}' | jq
```

기대 출력 예시:

```json
[
  {
    "address": "192.168.10.61",
    "type": "InternalIP"
  },
  {
    "address": "podpool1-worker-0.ocp420.aonsoft.demo.com",
    "type": "Hostname"
  }
]
```

노드 OS 내부 확인:

```bash
cat /etc/systemd/system/kubelet.service.d/20-nodenet.conf
ip route show default
ss -tlnp | grep -E ':(6443|22623|10250)'
```

기대값:

```text
KUBELET_NODE_IP=192.168.10.x
default via 192.168.10.1 dev bond0
```

---

## 1.7 IP 대역 설계

### 1.7.1 Infra + NAS Network

| 용도 | IP |
|---|---|
| Gateway | `192.168.10.1` |
| Bastion, DNS, Mirror Registry, HAProxy | `192.168.10.10` |
| API VIP / MCS VIP | `192.168.10.20` |
| Default Ingress VIP | `192.168.10.21` |
| Bootstrap, 설치 후 제거 | `192.168.10.30` |
| Master 0 | `192.168.10.31` |
| Master 1 | `192.168.10.32` |
| Master 2 | `192.168.10.33` |
| AP Worker 0 | `192.168.10.41` |
| AP Worker 1 | `192.168.10.42` |
| DB Worker 0 | `192.168.10.51` |
| DB Worker 1 | `192.168.10.52` |
| PodPool1 Worker 0, Infra IP | `192.168.10.61` |
| PodPool1 Worker 1, Infra IP | `192.168.10.62` |
| NAS VIP / NFS VIP | `192.168.10.70` |
| NAS Controller 1 | `192.168.10.71` |
| NAS Controller 2 | `192.168.10.72` |

---

### 1.7.2 External Service Network

| 용도 | IP |
|---|---|
| Gateway | `192.168.20.1` |
| Bastion Service NIC | `192.168.20.10` |
| Service Ingress VIP | `192.168.20.20` |
| Container Egress IP Pool | `192.168.20.30 ~ 192.168.20.49` |
| PodPool1 Worker 0, Service IP | `192.168.20.61` |
| PodPool1 Worker 1, Service IP | `192.168.20.62` |
| VM Service IP Pool | `192.168.20.100 ~ 192.168.20.199` |
| Reserved | `192.168.20.200 ~ 192.168.20.249` |

---

### 1.7.3 IP 설계의 학습 포인트

#### 1. VIP는 노드 IP와 분리

```text
API VIP: 192.168.10.20
Default Ingress VIP: 192.168.10.21
Service Ingress VIP: 192.168.20.20
```

PoC에서는 Bastion이 VIP를 처리하지만, 운영 전환 시에는 VIP만 외부 LB로 이전할 수 있습니다.

#### 2. 역할별 IP 그룹화

| 역할 | IP 대역 |
|---|---|
| Master | `.31 ~ .33` |
| AP Worker | `.41 ~ .42` |
| DB Worker | `.51 ~ .52` |
| PodPool1 Worker | `.61 ~ .62` |
| NAS | `.70 ~` |

#### 3. PodPool1은 두 대역에서 동일한 마지막 옥텟 사용

```text
podpool1-worker-0:
- Infra IP:   192.168.10.61
- Service IP: 192.168.20.61

podpool1-worker-1:
- Infra IP:   192.168.10.62
- Service IP: 192.168.20.62
```

#### 4. VM Service IP Pool은 별도 분리

```text
192.168.20.100 ~ 192.168.20.199
```

VM Guest OS가 사용할 IP는 노드 IP, VIP, egress IP pool과 겹치지 않도록 분리합니다.

---

## 1.8 노드별 NIC / Bond 설계

### 1.8.1 Master 노드

```text
bond0: Infra + NAS Network
  - IP: 192.168.10.31 / 32 / 33
  - default gateway: 192.168.10.1
  - 용도: API, DNS, MCS, Mirror Registry, NAS, etcd peer
```

Master는 Service망에 연결하지 않습니다.

이유:

- Master는 control plane 전용으로 유지
- Service망 트래픽이 Master에 도달하지 않도록 보안 경계 강화
- 업무 서비스 트래픽은 podpool1 또는 VM 노드에서 처리
- NIC 수와 구성 복잡도 감소

---

### 1.8.2 VM 전용 ap/db 노드

```text
bond0: Infra + NAS Network
  - IP: 192.168.10.41~42 / 192.168.10.51~52
  - default gateway: 192.168.10.1
  - 용도: API, DNS, MCS, Mirror, NAS, OVN node primary IP

bond1: VM Service Network
  - IP 없음
  - Linux bridge br-vm-svc 생성
  - NetworkAttachmentDefinition을 통해 VM Guest OS에 노출
  - VM Guest는 192.168.20.x 사용
```

구조:

```text
bond0, Infra
  └─ RHCOS host IP, 192.168.10.41
  └─ default gateway
  └─ kubelet / API / DNS / Mirror / NAS

bond1, Service
  └─ br-vm-svc, Linux bridge, IP 없음
       └─ NetworkAttachmentDefinition: vm-service-net
            └─ VM NIC, Guest OS가 192.168.20.x 사용
```

> **핵심 개념**  
> VM 노드의 Service망은 “노드가 사용하는 망”이 아니라 **VM에게 제공하는 망**입니다. RHCOS는 bond1에 IP를 갖지 않습니다. bond1은 VM Guest OS가 외부 Service망과 통신할 수 있도록 L2 통로 역할만 합니다.

---

### 1.8.3 Container 전용 podpool1 노드

```text
bond0: Infra + NAS Network
  - IP: 192.168.10.61 / 192.168.10.62
  - default gateway: 192.168.10.1
  - 용도: API, DNS, MCS, Mirror, NAS, OVN node primary IP

bond1: Container Service Network
  - IP: 192.168.20.61 / 192.168.20.62
  - 용도: Service IngressController, 컨테이너 업무 Egress
  - default gateway 없음
```

구조:

```text
bond0, Infra
  └─ RHCOS host IP, 192.168.10.61
  └─ default gateway, 192.168.10.1
  └─ kubelet / API / DNS / Mirror / NAS / MTV StorageClass

bond1, Service
  └─ RHCOS host IP, 192.168.20.61
  └─ Service IngressController
  └─ 컨테이너 업무 Egress 후보 경로
```

> **PodPool1만 두 NIC에 IP를 갖는 이유**  
> `bond0`은 OpenShift 노드 자체 통신용입니다. `bond1`은 업무 서비스의 Ingress/Egress 경로입니다. 따라서 podpool1은 두 NIC 모두에서 호스트가 능동적으로 트래픽을 주고받아야 하므로 두 NIC에 IP가 필요합니다.

---

### 1.8.4 NIC 구성 비교표

| 노드 역할 | bond0, Infra | bond1, Service | 비고 |
|---|---|---|---|
| Master | IP 있음, GW 있음 | 없음 | control plane 전용 |
| AP/DB Worker | IP 있음, GW 있음 | IP 없음, bridge만 | VM에 Service망 제공 |
| PodPool1 Worker | IP 있음, GW 있음 | IP 있음, GW 없음 | 업무 Ingress/Egress |
| Bastion | IP 있음, GW 있음 | IP 있음 | PoC에서 두 망 LB 처리 |

---

## 1.9 Default Gateway 정책

### 1.9.1 기본 정책

모든 노드의 default gateway는 Infra망에 둡니다.

| 대상 | Default Gateway |
|---|---|
| Master | `192.168.10.1` |
| AP/DB Worker | `192.168.10.1` |
| PodPool1 Worker | `192.168.10.1` |
| Bastion | `192.168.10.1` |

---

### 1.9.2 이유

| 이유 | 설명 |
|---|---|
| API/MCS 접근 안정성 | 노드가 API와 MCS에 안정적으로 접근 |
| DNS/Mirror/NAS 접근 안정성 | 핵심 인프라 서비스가 Infra망에 위치 |
| MachineConfig 안정성 | MCO 적용 중 노드 관리 통신 유지 |
| node InternalIP 안정성 | InternalIP가 Infra IP로 유지되기 쉬움 |
| 장애 분석 단순화 | 모든 노드의 기본 경로가 동일 |

---

### 1.9.3 Service망으로 가야 하는 트래픽

| 트래픽 | 처리 방법 |
|---|---|
| 업무 Pod의 외부 호출 | EgressIP 또는 정책 라우팅 |
| Service Ingress inbound | LB → podpool1 Service NIC |
| VM Guest OS의 외부 통신 | VM Guest OS default gateway, `192.168.20.1` |
| Service VIP 라우팅 | 외부 LB 또는 Bastion HAProxy |

> **흔한 실수**  
> 노드에 두 개의 default route를 설정하면, 라우팅 우선순위와 metric에 따라 트래픽 경로가 예측 불가능해질 수 있습니다. Service망으로 강제해야 하는 트래픽은 `ip rule`, EgressIP, 외부 방화벽 정책 등으로 명시적으로 처리합니다.

---

## 1.10 Ingress 설계

### 1.10.1 두 IngressController 분리 원칙

| 구분 | Default IngressController | Service IngressController |
|---|---|---|
| 용도 | Console, OAuth, 관리 Route | 사용자 업무 애플리케이션 Route |
| 배치 노드 | ap MCP, PoC | podpool1 MCP |
| 배포 방식 | HostNetwork, 80/443 | HostNetwork, 80/443 |
| 도메인 | `apps.ocp420.aonsoft.demo.com` | `svcapps.ocp420.aonsoft.demo.com` |
| VIP | `192.168.10.21` | `192.168.20.20` |
| 사용 망 | Infra | Service |

---

### 1.10.2 도메인 분리의 의미

Default IngressController의 도메인:

```text
*.apps.ocp420.aonsoft.demo.com
```

Service IngressController의 도메인:

```text
*.svcapps.ocp420.aonsoft.demo.com
```

DNS 등록:

```dns
*.apps.ocp420.aonsoft.demo.com      IN A   192.168.10.21
*.svcapps.ocp420.aonsoft.demo.com   IN A   192.168.20.20
```

이렇게 하면 DNS 단계에서 관리 트래픽과 업무 트래픽이 분리됩니다.

---

### 1.10.3 관리 Route 예시

| Namespace | Route 이름 | Host 예시 |
|---|---|---|
| `openshift-console` | `console` | `console-openshift-console.apps.ocp420.aonsoft.demo.com` |
| `openshift-authentication` | `oauth-openshift` | `oauth-openshift.apps.ocp420.aonsoft.demo.com` |
| `openshift-monitoring` | `thanos-querier` | `thanos-querier-openshift-monitoring.apps.ocp420.aonsoft.demo.com` |
| `openshift-monitoring` | `alertmanager-main` | `alertmanager-main-openshift-monitoring.apps.ocp420.aonsoft.demo.com` |

---

### 1.10.4 업무 Route 예시

```text
frontend.svcapps.ocp420.aonsoft.demo.com
api.svcapps.ocp420.aonsoft.demo.com
jboss-app.svcapps.ocp420.aonsoft.demo.com
```

업무 Route에는 다음 라벨을 사용합니다.

```yaml
metadata:
  labels:
    ingress: service
```

이 라벨을 통해 Service IngressController가 해당 Route를 처리하도록 구성합니다. 실제 `routeSelector` 구성은 이후 Part에서 다룹니다.

---

### 1.10.5 Default IngressController 배치 결정

#### PoC 환경

PoC에서는 Default IngressController를 `ap` 노드에 배치합니다.

```text
nodeSelector: node-role.kubernetes.io/ap=""
```

장점:

- 별도 infra 노드가 필요 없음
- podpool1의 Service Ingress와 포트 충돌 없음
- 관리 Ingress와 업무 Ingress를 노드 기준으로 분리 가능

단점:

- ap 노드가 완전한 VM 전용 노드는 아니게 됨
- router Pod가 VM 실행 노드와 공존함

#### 프로덕션 환경

프로덕션에서는 infra MCP를 별도로 구성하는 것을 권장합니다.

```text
Default IngressController → infra MCP
Monitoring / Logging / Registry → infra MCP
```

---

### 1.10.6 Service IngressController 배치

Service IngressController는 PoC와 프로덕션 모두 podpool1 MCP에 배치합니다.

```text
nodeSelector: node-role.kubernetes.io/podpool1=""
endpointPublishingStrategy: HostNetwork
domain: svcapps.ocp420.aonsoft.demo.com
```

이유:

- 컨테이너 업무 Pod가 podpool1에 배치됨
- Service망 NIC가 podpool1에 있음
- 업무 Ingress/Egress 흐름이 podpool1로 집중됨
- 운영자가 “컨테이너 업무 = podpool1”이라는 단일 규칙으로 이해 가능

---

## 1.11 Egress 설계

### 1.11.1 두 종류의 Egress

| Egress 종류 | 트래픽 | 망 | 메커니즘 |
|---|---|---|---|
| 관리 Egress | 노드의 이미지 pull, DNS, NTP, NAS | Infra | default gateway |
| 업무 Egress | 업무 Pod의 외부 시스템 호출 | Service | EgressIP 또는 정책 라우팅 |

---

### 1.11.2 관리 Egress

모든 노드의 default gateway가 Infra망에 있으므로 관리성 outbound 트래픽은 자동으로 Infra망을 사용합니다.

예:

- 노드 → Mirror Registry
- 노드 → DNS
- 노드 → NTP
- Pod → NAS VIP
- 노드 → API VIP

---

### 1.11.3 업무 Egress

업무 Pod가 외부 업무 시스템을 호출하는 트래픽은 Service망으로 분리하는 것이 목표입니다.

| 방식 | 설명 | 비고 |
|---|---|---|
| EgressIP | OVN-Kubernetes 기능 사용 | 1차 PoC 후보 |
| 정책 라우팅 | 노드 OS 레벨에서 `ip rule` 구성 | 대안 |
| 외부 방화벽/NAT | OpenShift 외부에서 정책 강제 | 보조 수단 |

> **PoC 권장**  
> 업무 Egress는 EgressIP를 1차 후보로 검증합니다. 단, secondary NIC인 podpool1의 Service NIC를 통한 EgressIP 동작은 PoC에서 반드시 검증해야 합니다. 실제 YAML과 검증 절차는 Part 6에서 다룹니다.

---

## 1.12 방화벽 및 포트 요구사항 요약

상세 포트 설계는 Part 2에서 다룹니다. Part 1에서는 전체 아키텍처 이해를 위한 핵심 포트만 정리합니다.

| 방향 | 포트 | 프로토콜 | 용도 |
|---|---:|---|---|
| Client/Admin → API VIP | 6443 | TCP | Kubernetes API |
| Node → MCS VIP | 22623 | TCP | Machine Config Server |
| Node ↔ Node | 6081 | UDP | OVN-Kubernetes Geneve |
| Bastion → Router | 80/443 | TCP | Ingress 트래픽 |
| Bastion → Router | 1936 | TCP | Router health check |
| Node → DNS | 53 | TCP/UDP | DNS |
| Node → Mirror Registry | 환경별 | TCP | Image pull |
| Node → NTP | 123 | UDP | Time sync |
| Node ↔ NAS | 2049 등 | TCP/UDP | NFS, NAS StorageClass |

> **중요**  
> OVN-Kubernetes를 사용할 경우 모든 노드 간 UDP 6081 통신이 가능해야 합니다. 이 통신이 막히면 Pod 간 통신이 실패할 수 있습니다.

---

## 1.13 PoC vs 프로덕션 차이

| 항목 | PoC | 프로덕션 권장 |
|---|---|---|
| infra MCP | 없음, ap에 default Ingress 배치 | infra MCP 신설 |
| Default IngressController | ap MCP | infra MCP |
| Service IngressController | podpool1 MCP | podpool1 또는 업무 router 전용 노드 |
| Monitoring / Logging / Registry | 기본 위치 또는 최소 구성 | infra MCP로 이동 |
| Default Ingress LB | Bastion HAProxy | 관리망 LB 또는 HAProxy 이중화 |
| Service Ingress LB | Bastion HAProxy | 별도 물리 L4/L7 |
| Bastion NIC | Infra + Service | Infra 중심, Service LB 분리 |
| NAS Network | Infra와 통합 | 별도 NAS망 분리 검토 |
| LiveMigration Network | Infra와 공유 | 별도 migration network 검토 |
| Monitoring Storage | emptyDir 또는 NFS | block storage 권장 |
| LB HA | 단일 Bastion | LB 장비 이중화 |

---

### 1.13.1 전환 시 영향이 큰 항목

#### 1. infra MCP 신설

```text
Default IngressController
Monitoring
Logging
Image Registry
```

위 구성요소를 infra MCP로 이동합니다.

#### 2. Service LB 분리

PoC에서는 Bastion HAProxy가 Service Ingress VIP를 처리합니다.

프로덕션에서는 다음처럼 분리합니다.

```text
Service Ingress VIP → 물리 L4/L7
Backend → podpool1 Service IP
```

#### 3. NAS망 분리

PoC에서는 NAS가 Infra와 통합되어 있습니다. 프로덕션에서는 NAS 트래픽 경합을 줄이기 위해 별도 NAS망 분리를 검토합니다.

---

## 1.14 다음 단계

Part 1에서 정의한 설계를 바탕으로 다음 파트를 진행합니다.

| Part | 문서 | 주요 내용 |
|---|---|---|
| Part 2 | Bastion 및 Load Balancer | DNS, HAProxy, VIP, health check, bootstrap 제거 |
| Part 3 | 미러링 및 설치 | oc-mirror v2, install-config, RHCOS 설치 |
| Part 4 | Day-2 MCP 및 MachineConfig | MCP 분리, chrony, kdump, multipath, 인증서 |
| Part 5 | Operator 및 Virtualization | NMState, NNCP, NAD, Virtualization, StorageClass |
| Part 6 | 시나리오 테스트 | MTV, FAR/vBMC, Tekton, GitOps, JBoss, 트러블슈팅 |

---

## 1.15 Part 1 학습 점검

다음 질문에 답할 수 있다면 Part 1을 충분히 학습한 것입니다.

1. 본 PoC가 두 개의 외부 네트워크를 분리한 이유는 무엇인가?
2. `machineNetwork`에 Service망 `192.168.20.0/24`를 추가하면 어떤 문제가 발생할 수 있는가?
3. podpool1 노드는 왜 두 NIC에 모두 IP를 가지는데 ap/db 노드의 bond1에는 IP가 없는가?
4. 모든 노드의 default gateway를 Infra망에 두는 이유 세 가지를 설명하시오.
5. Default IngressController와 Service IngressController를 분리하는 이유는 무엇인가?
6. `*.apps.ocp420.aonsoft.demo.com`과 `*.svcapps.ocp420.aonsoft.demo.com`을 분리하는 이유는 무엇인가?
7. PoC에서 Default IngressController를 ap에 두는 결정의 trade-off는 무엇인가?
8. 업무 Pod의 outbound를 Service망으로 강제하는 대표적인 방법은 무엇인가?
9. PoC에서 프로덕션으로 전환할 때 가장 큰 변경 세 가지는 무엇인가?
10. OVN-Kubernetes 사용 시 노드 간 어떤 포트가 반드시 열려 있어야 하는가?

---

## 1.16 Part 1 요약

Part 1의 핵심은 다음과 같습니다.

```text
Infra망은 관리 트래픽 전용이다.
Service망은 업무 트래픽 전용이다.
NAS는 PoC에서 Infra망과 통합한다.
machineNetwork는 192.168.10.0/24만 사용한다.
node InternalIP는 반드시 Infra IP가 되어야 한다.
Master는 Service망에 연결하지 않는다.
ap/db는 VM 전용 노드이며 bond1은 VM에게 Service망을 제공한다.
podpool1은 Infra IP와 Service IP를 모두 가지며 컨테이너 업무 트래픽을 처리한다.
Default Ingress는 ap에 배치하고 Infra망에서 접근한다.
Service Ingress는 podpool1에 배치하고 Service망에서 접근한다.
PoC에서는 Bastion HAProxy가 모든 VIP를 처리하지만, 프로덕션에서는 Service LB와 infra 워크로드를 분리한다.
```

---

*Part 1 끝. 다음은 Part 2 — Bastion 및 Load Balancer 구성입니다.*
