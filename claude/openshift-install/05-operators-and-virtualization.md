# Part 5. Operator 설치 및 OpenShift Virtualization

> **이 파트의 목적**
> Part 4에서 노드 그룹과 OS 설정이 분리되었다면, 본 파트에서는 그 위에 동작할 Operator들을 설치하고 워크로드 실행 환경을 구성합니다. 핵심은 다음과 같습니다.
>
> 1. Operator 의존성을 인지하고 올바른 순서로 설치
> 2. NMState로 VM 노드의 Service NIC를 Linux bridge로 변환 (Part 1.6.2의 설계를 실제 구현)
> 3. OpenShift Virtualization을 설치하고 ap/db에만 VM이 배치되도록 nodePlacement 구성
> 4. NetworkAttachmentDefinition(NAD)으로 VM에 Service망 노출
> 5. NAS NFS StorageClass 구성
> 6. Default IngressController(ap)와 Service IngressController(podpool1) 분리 구성
>
> 이 파트를 학습하고 나면 OLM 기반 Operator 설치, CNV(Container-Native Virtualization)의 컴포넌트 구조, 그리고 외부 네트워크를 VM에 노출시키는 메커니즘을 설명할 수 있어야 합니다.

---

## 5.1 Operator Lifecycle Manager 개요

### 5.1.1 OLM이란

**OLM(Operator Lifecycle Manager)**은 OpenShift에서 Operator를 설치, 업그레이드, 관리하는 메타 Operator입니다. OperatorHub UI에서 Operator를 "Install" 버튼으로 클릭하면 뒤에서 OLM이 동작합니다.

OLM이 다루는 핵심 리소스:

| 리소스 | 역할 |
|---|---|
| `CatalogSource` | Operator 목록 제공 (gRPC 서버) |
| `PackageManifest` | 카탈로그에서 추출된 Operator 메타데이터 |
| `OperatorGroup` | namespace 단위 Operator 권한 범위 정의 |
| `Subscription` | "이 Operator를 설치/유지해 달라"는 사용자 의도 |
| `InstallPlan` | Subscription에 따른 실제 설치 계획 |
| `ClusterServiceVersion (CSV)` | 설치된 Operator 한 버전의 정의 |

### 5.1.2 OLM 흐름

```
사용자가 Subscription 생성
   ↓
OLM이 CatalogSource에서 패키지 정보 조회
   ↓
의존성 해결 후 InstallPlan 생성
   ↓
InstallPlan이 ClusterServiceVersion(CSV) 배포
   ↓
CSV가 Operator Deployment, RBAC, CRD 생성
   ↓
Operator Pod 동작 시작
   ↓
사용자가 Operator의 CR (예: HyperConverged) 생성
   ↓
Operator가 CR을 보고 실제 워크로드 배포 (예: virt-launcher Pod)
```

### 5.1.3 폐쇄망에서의 OLM

Part 3.10에서 다음 작업을 이미 완료했습니다.
- oc-mirror가 생성한 `cs-redhat-operator-index-v4-20.yaml` CatalogSource 적용
- `OperatorHub` 리소스에서 `disableAllDefaultSources: true` 설정

이로 인해 클러스터는 외부 카탈로그(`registry.redhat.io`)를 시도하지 않고 미러 카탈로그만 사용합니다.

```bash
# 사용 가능한 카탈로그 확인
oc get catalogsource -A
```

기대:
```
NAMESPACE              NAME                              DISPLAY                     TYPE   PUBLISHER   AGE
openshift-marketplace  cs-redhat-operator-index-v4-20    Red Hat Operators (Mirror)  grpc   Red Hat     2h
```

```bash
# 사용 가능한 패키지 목록 (미러된 것만)
oc get packagemanifest -n openshift-marketplace
```

본 PoC에서 미러링한 8개 Operator가 표시되어야 합니다.

---

## 5.2 Operator 설치 순서

### 5.2.1 의존성 기반 권장 순서

본 PoC에서 설치할 Operator들의 의존성과 권장 순서:

```
[1단계: 인프라 네트워크]
   └─ Kubernetes NMState Operator
        ↓ (VM bridge 생성에 필요)
[2단계: Virtualization]
   └─ OpenShift Virtualization Operator
        ↓ (VM 워크로드 실행 기반)
[3단계: 스토리지]
   └─ NFS StorageClass (별도 Operator 또는 직접 구성)
        ↓ (MTV 대상 PVC 제공)
[4단계: 마이그레이션]
   └─ MTV (Migration Toolkit for Virtualization) Operator
        ↓
[5단계: 컨테이너 워크로드 플랫폼]
   ├─ OpenShift Pipelines (Tekton)
   ├─ OpenShift GitOps
   └─ Service Mesh
        ↓
[6단계: Fencing]
   ├─ Node Health Check Operator
   └─ Fence Agents Remediation Operator
```

### 5.2.2 왜 이 순서인가

**1단계 (NMState 먼저):**
- OpenShift Virtualization의 VM이 외부 네트워크와 통신하려면 호스트의 NIC를 Linux bridge로 만들어야 함
- NMState가 이 작업을 NodeNetworkConfigurationPolicy(NNCP)로 선언적으로 수행
- Virtualization을 먼저 설치하면 VM이 갈 곳이 없음

**2단계 (Virtualization 다음):**
- HyperConverged CR이 NMState가 만든 bridge 위에서 NetworkAttachmentDefinition(NAD)을 사용
- virt-launcher Pod, CDI(Containerized Data Importer) 등 핵심 컴포넌트 배포

**3단계 (스토리지):**
- VM 디스크 저장용 PVC가 필요
- MTV 마이그레이션 대상 StorageClass 필요

**4단계 (MTV):**
- Virtualization과 StorageClass가 모두 ready 상태에서 설치
- 의존성을 만족하지 못하면 MTV의 ForkliftController가 ready되지 않음

**5단계 (앱 플랫폼):**
- 컨테이너 워크로드 검증용
- 서로 독립적이므로 어느 것을 먼저 설치해도 무방

**6단계 (Fencing):**
- 클러스터가 안정화된 후 fencing 정책 적용
- NHC가 FAR를 호출하는 구조이므로 NHC 먼저 설치

> **순서를 어겨도 동작은 하나 트러블슈팅이 복잡해진다**
> 의존성 위반 시 Operator가 "Pending" 또는 "Failed" 상태로 멈추는데, 정확히 어떤 의존성이 누락되었는지 메시지가 불명확할 때가 있습니다. 권장 순서를 따르면 각 단계마다 검증 후 다음으로 넘어갈 수 있어 문제 발생 시 원인 추적이 쉬워집니다.

---

## 5.3 NMState Operator 설치

### 5.3.1 NMState의 역할

NMState는 노드의 네트워크 상태를 선언적으로 관리하는 도구입니다. OpenShift에서는 NMState Operator가 다음 리소스를 제공합니다.

| 리소스 | 역할 |
|---|---|
| `NodeNetworkState (NNS)` | 노드의 현재 네트워크 상태 조회 (읽기 전용) |
| `NodeNetworkConfigurationPolicy (NNCP)` | 원하는 네트워크 구성 선언 |
| `NodeNetworkConfigurationEnactment (NNCE)` | NNCP가 노드에 적용된 결과 |

본 PoC에서 NMState를 사용하는 목적:

1. **ap/db 노드의 bond1을 Linux bridge로 변환** — VM이 Service망에 직접 연결되도록
2. **podpool1 노드의 bond1 상태 확인** — 호스트 IP가 정상 할당되었는지 검증
3. **향후 네트워크 변경의 선언적 관리** — MachineConfig보다 가벼운 네트워크 변경

### 5.3.2 Subscription 생성

```bash
mkdir -p ~/openshift-day2/operators
cd ~/openshift-day2/operators
```

```bash
cat > 01-nmstate-operator.yaml <<'EOF'
---
# 1. namespace
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-nmstate
  labels:
    openshift.io/cluster-monitoring: "true"
---
# 2. OperatorGroup
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-nmstate
  namespace: openshift-nmstate
spec:
  targetNamespaces:
  - openshift-nmstate
---
# 3. Subscription
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kubernetes-nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubernetes-nmstate-operator
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
EOF
```

### 5.3.3 적용 및 확인

```bash
oc apply -f 01-nmstate-operator.yaml

# 설치 진행 모니터링
watch 'oc get csv -n openshift-nmstate; echo "---"; oc get pods -n openshift-nmstate'
```

기대 (수 분 후):
```
NAME                                           DISPLAY                       VERSION    PHASE
kubernetes-nmstate-operator.4.20.0-x.y         Kubernetes NMState Operator   4.20.0     Succeeded
```

```bash
# NMState 인스턴스 CR 생성 (Operator 동작 시작)
cat > 02-nmstate-instance.yaml <<'EOF'
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
EOF

oc apply -f 02-nmstate-instance.yaml
```

이 CR을 적용하면 NMState Operator가 다음을 배포합니다:
- `nmstate-handler` DaemonSet (모든 노드에 nmstate agent)
- `nmstate-webhook` Deployment (CR 검증)

```bash
# 모든 노드에 handler 배포 확인
oc get ds -n openshift-nmstate
```

기대:
```
NAME              DESIRED   CURRENT   READY   ...
nmstate-handler   9         9         9       ...
```

### 5.3.4 NodeNetworkState 확인

설치가 완료되면 각 노드의 현재 네트워크 상태를 NNS로 확인할 수 있습니다.

```bash
# 노드별 NNS 조회
oc get nns

# 특정 노드의 상세 상태
oc get nns ap-worker-0 -o yaml | yq '.status.currentState' | head -100
```

이 출력에는 노드의 모든 NIC, IP, 라우팅 등이 들어 있어 향후 트러블슈팅에 매우 유용합니다.

---

## 5.4 VM Service Bridge 구성 (NNCP)

### 5.4.1 구성 목표

Part 1.6.2에서 정의한 ap/db 노드의 NIC 구조를 실제로 구현합니다.

```
구현 전 (RHCOS 기본):
ap-worker-0
├─ ens192 (Infra): 192.168.10.41/24
└─ ens224 (Service): IP 없음, NetworkManager profile은 있지만 활용 안 됨

구현 후 (NNCP 적용 후):
ap-worker-0
├─ ens192 (Infra): 192.168.10.41/24 (변경 없음)
└─ ens224 (Service): Linux bridge "br-vm-svc"의 port로 attach
    └─ br-vm-svc (bridge, IP 없음)
         └─ VM이 NetworkAttachmentDefinition을 통해 연결됨
```

### 5.4.2 NNCP YAML 작성

```bash
cat > 03-nncp-vm-service-bridge.yaml <<'EOF'
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vm-service-bridge-ap
spec:
  # ap MCP 노드만 대상
  nodeSelector:
    node-role.kubernetes.io/ap: ""

  desiredState:
    interfaces:
    # 1. Linux bridge 생성
    - name: br-vm-svc
      description: "VM Service Network bridge"
      type: linux-bridge
      state: up
      ipv4:
        enabled: false      # bridge에 호스트 IP 부여 안 함
      ipv6:
        enabled: false
      bridge:
        options:
          stp:
            enabled: false  # PoC는 단순화, STP 불필요
        port:
        - name: ens224      # Service NIC를 bridge port로 추가

    # 2. ens224는 bridge에 attach되므로 자체 IP/라우팅 제거
    - name: ens224
      type: ethernet
      state: up
      ipv4:
        enabled: false
      ipv6:
        enabled: false
EOF

# DB도 동일 구성
cat > 04-nncp-vm-service-bridge-db.yaml <<'EOF'
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vm-service-bridge-db
spec:
  nodeSelector:
    node-role.kubernetes.io/db: ""
  desiredState:
    interfaces:
    - name: br-vm-svc
      description: "VM Service Network bridge"
      type: linux-bridge
      state: up
      ipv4:
        enabled: false
      ipv6:
        enabled: false
      bridge:
        options:
          stp:
            enabled: false
        port:
        - name: ens224
    - name: ens224
      type: ethernet
      state: up
      ipv4:
        enabled: false
      ipv6:
        enabled: false
EOF
```

### 5.4.3 NNCP 핵심 옵션 해설

**`nodeSelector`**

```yaml
nodeSelector:
  node-role.kubernetes.io/ap: ""
```

이 NNCP는 ap 라벨이 있는 노드에만 적용됩니다. 결과적으로 ap-worker-0과 ap-worker-1만 bridge가 만들어지며, podpool1과 master는 영향 없음.

**`bridge.options.stp.enabled: false`**

STP(Spanning Tree Protocol)는 L2 루프 방지에 사용되지만, 단일 NIC를 bridge에 연결하는 본 PoC에서는 불필요하며 활성화 시 bridge가 forwarding 상태가 되기까지 30~50초 지연이 발생합니다. **PoC에서는 비활성화 권장.**

**`port: [{name: ens224}]`**

bridge의 멤버 NIC. ens224 하나만 추가하여 단순한 "NIC pass-through" 구조를 만듭니다.

**ens224의 IP 비활성화**

bridge가 만들어진 후 ens224 자체에는 IP가 없어야 합니다. IP가 ens224에 남아있으면 라우팅이 충돌합니다.

> **흔한 실수: bridge에 IP를 부여**
> bridge에 호스트 IP를 부여하면 호스트가 그 망에서 통신 가능한 노드가 됩니다. 그러면 Part 1.6.2의 "AP/DB는 Service망을 호스트가 사용하지 않는다"는 원칙이 깨집니다. **bridge에 IP를 부여하지 않는 것이 핵심**입니다.

### 5.4.4 NNCP 적용

```bash
oc apply -f 03-nncp-vm-service-bridge.yaml
oc apply -f 04-nncp-vm-service-bridge-db.yaml

# 진행 상태 확인 (NNCE: 노드별 적용 결과)
oc get nnce
```

기대 (적용 진행 중):
```
NAME                                          STATUS
ap-worker-0.vm-service-bridge-ap              Progressing
ap-worker-1.vm-service-bridge-ap              Progressing
db-worker-0.vm-service-bridge-db              Progressing
db-worker-1.vm-service-bridge-db              Progressing
```

완료 후:
```
ap-worker-0.vm-service-bridge-ap              Available
ap-worker-1.vm-service-bridge-ap              Available
db-worker-0.vm-service-bridge-db              Available
db-worker-1.vm-service-bridge-db              Available
```

NNCP 자체 상태:

```bash
oc get nncp
```

기대:
```
NAME                       STATUS      REASON
vm-service-bridge-ap       Available   SuccessfullyConfigured
vm-service-bridge-db       Available   SuccessfullyConfigured
```

### 5.4.5 검증

```bash
# ap-worker-0의 bridge 확인
oc debug node/ap-worker-0 -- chroot /host ip link show br-vm-svc
# 기대: br-vm-svc 인터페이스 출력 (UP)

oc debug node/ap-worker-0 -- chroot /host bridge link
# 기대: ens224가 br-vm-svc의 멤버로 표시

# bridge에 IP가 없는지 확인 (반드시 IP 없어야 함)
oc debug node/ap-worker-0 -- chroot /host ip addr show br-vm-svc
# 기대: inet 라인이 없음
```

### 5.4.6 흔한 실수와 대응

| 증상 | 원인 | 해결 |
|---|---|---|
| NNCE가 Progressing 멈춤 | NIC 이름이 노드별로 다름 (ens224 vs eth1) | 노드별 NNS로 실제 NIC 이름 확인 후 NNCP 분리 |
| NNCE가 Failing, 노드 SSH 끊김 | 잘못된 NIC를 bridge에 attach | NMState가 1분 후 자동 rollback |
| bridge는 있는데 VM 통신 불가 | 외부 스위치 포트의 VLAN/Trunk 설정 누락 | 스위치 측 VLAN 허용 필요 |

> **NMState의 자동 rollback 기능**
> NNCP 적용 후 노드가 1분 이내에 API 서버와 통신을 유지하지 못하면 NMState가 자동으로 원래 상태로 되돌립니다. 이는 잘못된 네트워크 변경으로 노드가 영구 격리되는 것을 방지하는 안전 장치입니다.
> 그래서 NNCP 적용은 위험도가 낮으며, 학습 단계에서 시도해보기 좋은 영역입니다.

---

## 5.5 OpenShift Virtualization 설치

### 5.5.1 OpenShift Virtualization 개요

OpenShift Virtualization(OCP-V)은 KubeVirt 기반으로 Kubernetes 위에서 VM을 실행하는 솔루션입니다. 주요 컴포넌트:

| 컴포넌트 | 역할 | 배포 위치 |
|---|---|---|
| `virt-operator` | Virtualization 자체 관리 | infra (또는 master) |
| `virt-api` | VirtualMachine API 서버 | infra |
| `virt-controller` | VM 라이프사이클 컨트롤러 | infra |
| `virt-handler` | 노드별 VM 관리 DaemonSet | 모든 worker (VM 가능 노드) |
| `virt-launcher` | 각 VM당 1개 Pod, QEMU 실행 | VM이 배치된 worker |
| `cdi-operator` | Containerized Data Importer 관리 | infra |
| `cdi-uploadproxy/importer` | VM 디스크 이미지 import | worker |
| `hostpath-provisioner` | 로컬 스토리지 (선택) | worker |

### 5.5.2 Subscription 생성

```bash
cat > 05-virtualization-operator.yaml <<'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cnv
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
  - openshift-cnv
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubevirt-hyperconverged
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
EOF

oc apply -f 05-virtualization-operator.yaml

# 설치 진행
watch 'oc get csv -n openshift-cnv'
```

기대 (수 분 후):
```
NAME                                          DISPLAY                              VERSION    PHASE
kubevirt-hyperconverged-operator.v4.20.0      OpenShift Virtualization             4.20.0     Succeeded
```

### 5.5.3 HyperConverged CR 작성

OpenShift Virtualization의 모든 컴포넌트를 한 번에 배포하는 단일 CR입니다.

```bash
cat > 06-hyperconverged.yaml <<'EOF'
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:

  # ===========================================
  # infra 컴포넌트 배치 (control plane용)
  # PoC: master에 배치 (infra MCP 없음)
  # 프로덕션: infra MCP에 배치 권장
  # ===========================================
  infra:
    nodePlacement:
      nodeSelector:
        node-role.kubernetes.io/master: ""
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"

  # ===========================================
  # workload 컴포넌트 배치 (virt-launcher, CDI 등)
  # ap, db 노드에 배치 (VM 실행 대상)
  # ===========================================
  workloads:
    nodePlacement:
      nodeSelector:
        # ap 또는 db 노드 모두 매칭하려면 affinity 사용
        # 여기서는 단순화를 위해 nodeSelector 대신 affinity 사용
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/ap
                operator: Exists
            - matchExpressions:
              - key: node-role.kubernetes.io/db
                operator: Exists

  # ===========================================
  # 기능 옵션
  # ===========================================
  featureGates:
    enableCommonBootImageImport: true   # 기본 OS template 자동 import
    deployTektonTaskResources: false    # 별도 설치할 것이므로 false
    deployKubeSecondaryDNS: false

  # ===========================================
  # LiveMigration 설정
  # ===========================================
  liveMigrationConfig:
    completionTimeoutPerGiB: 800
    parallelMigrationsPerCluster: 5
    parallelOutboundMigrationsPerNode: 2
    progressTimeout: 150
    # network: 별도 migration network 지정 (PoC: 미지정, Infra 사용)

  # ===========================================
  # 인증서 자동 갱신
  # ===========================================
  certConfig:
    ca:
      duration: 48h0m0s
      renewBefore: 24h0m0s
    server:
      duration: 24h0m0s
      renewBefore: 12h0m0s
EOF

oc apply -f 06-hyperconverged.yaml
```

### 5.5.4 HyperConverged 옵션 해설

**`infra.nodePlacement`**

```yaml
infra:
  nodePlacement:
    nodeSelector:
      node-role.kubernetes.io/master: ""
    tolerations:
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
      effect: "NoSchedule"
```

`virt-operator`, `virt-api`, `virt-controller`, `cdi-operator`, `cdi-apiserver` 같은 control plane 성격 Pod들을 어디에 배치할지 지정합니다.

**PoC 선택지:**

| 옵션 | 배치 | 장단점 |
|---|---|---|
| A: master에 배치 | `master` taint를 toleration | master 자원 사용, 단 별도 노드 불필요 |
| B: podpool1에 배치 | `podpool1` 선택 | podpool1 자원 사용 |
| C: 지정 안 함 (기본) | worker 전체에 분산 | ap/db에 infra Pod 섞임 (격리 약화) |

본 PoC는 **옵션 A (master 배치)**를 권장합니다. infra 컴포넌트의 자원 부하가 크지 않고, master의 여유 자원을 활용할 수 있으며, ap/db/podpool1 노드를 워크로드 전용으로 깔끔하게 유지할 수 있습니다.

> **프로덕션에서는 infra MCP 배치 권장**
> 프로덕션에서는 infra MCP를 신설하고 `infra.nodePlacement.nodeSelector: { node-role.kubernetes.io/infra: "" }`로 분리합니다. master는 control plane 전용으로 유지.

**`workloads.nodePlacement.affinity`**

```yaml
workloads:
  nodePlacement:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: node-role.kubernetes.io/ap
              operator: Exists
          - matchExpressions:
            - key: node-role.kubernetes.io/db
              operator: Exists
```

`virt-handler` DaemonSet, `virt-launcher` Pod, CDI importer 등 VM 실행에 직접 관여하는 컴포넌트를 ap 또는 db 노드에만 배치합니다.

`nodeSelector`는 라벨 하나만 지정 가능하지만, `affinity`의 여러 `nodeSelectorTerms`는 OR 조건으로 동작하여 "ap 또는 db"를 표현할 수 있습니다.

> **podpool1에 virt-handler가 가지 않아야 한다**
> virt-handler는 그 노드에서 VM 실행을 가능하게 하는 컴포넌트입니다. podpool1에 virt-handler가 가면 누군가 실수로 VM을 podpool1에 띄울 수 있게 됩니다. 컨테이너 격리 원칙 유지를 위해 podpool1에서 virt-handler 배제는 핵심입니다.

**`liveMigrationConfig`**

LiveMigration은 VM을 다운타임 없이 다른 노드로 옮기는 기능입니다.

| 옵션 | 의미 |
|---|---|
| `parallelMigrationsPerCluster: 5` | 클러스터 전체에서 동시 마이그레이션 수 |
| `parallelOutboundMigrationsPerNode: 2` | 한 노드에서 동시에 떠나는 VM 수 |
| `completionTimeoutPerGiB: 800` | VM 디스크 1GiB당 마이그레이션 완료 timeout(초) |
| `network` (미지정) | 마이그레이션 트래픽 네트워크 (PoC는 Infra 공유) |

**PoC에서는 `network`를 지정하지 않아 Infra망에서 LiveMigration이 수행됩니다.** 프로덕션에서는 별도 migration network NAD를 만들어 지정하는 것을 권장.

### 5.5.5 설치 진행 모니터링

```bash
watch 'oc get pods -n openshift-cnv'
```

기대 (10~20분 후):
```
NAME                                    READY   STATUS    RESTARTS   AGE
cdi-apiserver-XXXXX                     1/1     Running   0          5m
cdi-deployment-XXXXX                    1/1     Running   0          5m
cdi-operator-XXXXX                      1/1     Running   0          15m
cdi-uploadproxy-XXXXX                   1/1     Running   0          5m
cluster-network-addons-operator-XXXXX   2/2     Running   0          15m
hco-operator-XXXXX                      1/1     Running   0          15m
hostpath-provisioner-operator-XXXXX     1/1     Running   0          15m
hyperconverged-cluster-operator-XXXXX   1/1     Running   0          15m
hyperconverged-cluster-webhook-XXXXX    1/1     Running   0          15m
kubemacpool-cert-manager-XXXXX          1/1     Running   0          5m
kubemacpool-mac-controller-XXXXX        1/1     Running   0          5m
kubevirt-apiserver-proxy-XXXXX          1/1     Running   0          5m
ssp-operator-XXXXX                      1/1     Running   0          15m
virt-api-XXXXX                          1/1     Running   0          10m
virt-controller-XXXXX                   1/1     Running   0          10m
virt-handler-XXXXX                      1/1     Running   0          5m
virt-handler-XXXXX                      1/1     Running   0          5m
virt-handler-XXXXX                      1/1     Running   0          5m
virt-handler-XXXXX                      1/1     Running   0          5m
virt-operator-XXXXX                     1/1     Running   0          15m
```

### 5.5.6 nodePlacement 검증

**핵심: virt-handler가 ap/db에만 있고 podpool1에는 없는지 확인**

```bash
oc get pods -n openshift-cnv -l kubevirt.io=virt-handler -o wide
```

기대:
```
NAME                READY   STATUS    NODE                IP
virt-handler-XXXXX  1/1     Running   ap-worker-0         10.128.x.x
virt-handler-XXXXX  1/1     Running   ap-worker-1         10.128.x.x
virt-handler-XXXXX  1/1     Running   db-worker-0         10.128.x.x
virt-handler-XXXXX  1/1     Running   db-worker-1         10.128.x.x
```

**4개만 표시되어야 함.** podpool1-worker-0/1에 virt-handler가 있다면 nodePlacement가 잘못 적용된 것입니다.

```bash
# HyperConverged CR의 상태 확인
oc get hyperconverged kubevirt-hyperconverged -n openshift-cnv -o jsonpath='{.status.conditions}' | jq
```

기대: `Available=True`, `Progressing=False`

---

## 5.6 NetworkAttachmentDefinition (NAD)

### 5.6.1 NAD의 역할

NAD는 OpenShift에서 VM 또는 Pod가 secondary network에 연결할 수 있도록 정의하는 리소스입니다. NMState가 만든 bridge를 VM이 사용하려면 NAD가 중간 매개체 역할을 합니다.

```
VM Guest OS
   ↓ (eth0)
NAD: vm-service-net
   ↓
Linux bridge: br-vm-svc
   ↓
호스트 NIC: ens224
   ↓
외부 Service망 (192.168.20.0/24)
```

### 5.6.2 NAD YAML 작성

```bash
cat > 07-nad-vm-service.yaml <<'EOF'
---
# VM이 사용할 namespace에 NAD 생성
apiVersion: v1
kind: Namespace
metadata:
  name: vm-workload
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vm-service-net
  namespace: vm-workload
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br-vm-svc
spec:
  config: |
    {
      "cniVersion": "0.4.0",
      "name": "vm-service-net",
      "type": "cnv-bridge",
      "bridge": "br-vm-svc",
      "macspoofchk": true,
      "ipam": {}
    }
EOF
```

### 5.6.3 NAD 옵션 해설

**`type: cnv-bridge`**

OpenShift Virtualization 전용 CNI 플러그인. 일반 `bridge` 플러그인보다 VM 친화적 기능을 제공:
- MAC spoof check
- VLAN tagging
- IP 할당 없이 L2 pass-through

**`bridge: br-vm-svc`**

5.4에서 만든 Linux bridge 이름. **NMState NNCP에서 만든 bridge 이름과 일치해야 합니다.**

**`ipam: {}` (빈 값)**

VM의 IP 할당을 NAD에서 하지 않는다는 의미. VM Guest OS가 자체적으로 IP를 설정 (수동 또는 외부 DHCP).

**대안: NAD에서 IP 할당**

만약 PoC에서 외부 DHCP가 없고 VM에 자동 IP를 부여하고 싶다면:

```yaml
spec:
  config: |
    {
      "cniVersion": "0.4.0",
      "name": "vm-service-net",
      "type": "cnv-bridge",
      "bridge": "br-vm-svc",
      "macspoofchk": true,
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.20.100/24",
        "range_start": "192.168.20.100",
        "range_end": "192.168.20.199",
        "gateway": "192.168.20.1"
      }
    }
```

`whereabouts`는 OpenShift 내장 IPAM(IP Address Management) CNI로, range에서 자동으로 IP를 할당합니다.

> **본 PoC에서는 IPAM 없이 진행 권장**
> VM Guest OS 내부에서 수동 IP 설정 또는 cloud-init으로 IP 부여. IPAM CNI를 사용하면 VM 외부에서 IP를 통제할 수 있어 운영이 편하지만, PoC 단계에서는 단순성을 위해 미설정.

### 5.6.4 NAD 적용 및 검증

```bash
oc apply -f 07-nad-vm-service.yaml

# 확인
oc get net-attach-def -n vm-workload
```

기대:
```
NAME             AGE
vm-service-net   1m
```

---

## 5.7 VM 테스트 (간단한 VM 생성)

NMState bridge와 NAD가 동작하는지 검증하기 위해 간단한 VM을 만들어 보겠습니다.

### 5.7.1 VM YAML 작성

```bash
cat > 08-test-vm.yaml <<'EOF'
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: test-vm-fedora
  namespace: vm-workload
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: test-vm-fedora
    spec:
      domain:
        cpu:
          cores: 2
        memory:
          guest: 2Gi
        devices:
          disks:
          - name: rootdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}        # Pod network (관리용)
          - name: vm-service
            bridge: {}            # Service network (업무용)
      networks:
      - name: default
        pod: {}
      - name: vm-service
        multus:
          networkName: vm-service-net    # 위에서 만든 NAD
      # ap 노드에만 배치
      nodeSelector:
        node-role.kubernetes.io/ap: ""
      volumes:
      - name: rootdisk
        containerDisk:
          image: quay.io/containerdisks/fedora:39
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
            ssh_pwauth: True
            runcmd:
            # Service NIC를 수동 설정
            - nmcli con add type ethernet con-name vm-service ifname eth1 ip4 192.168.20.150/24 gw4 192.168.20.1
            - nmcli con up vm-service
EOF

oc apply -f 08-test-vm.yaml
```

> **PoC 환경에서 containerDisk 이미지 미러링 필요**
> 위 예시는 `quay.io/containerdisks/fedora:39`를 사용합니다. 폐쇄망에서는 이 이미지를 별도로 미러링하거나, 미리 다운로드한 ISO/qcow2를 PVC에 import하는 방식으로 대체해야 합니다.
> 실제 MTV 마이그레이션 검증에서는 외부 VM을 가져오므로 이 testVM은 NMState/NAD 동작 검증용 단순 예시로 보시면 됩니다.

### 5.7.2 VM 동작 검증

```bash
# VM 상태
oc get vm -n vm-workload
# 기대: test-vm-fedora  AGE  Running=True   Ready=True

oc get vmi -n vm-workload
# vmi(VirtualMachineInstance)가 Running 상태

oc get vmi test-vm-fedora -n vm-workload -o wide
# NODE 컬럼이 ap-worker-0 또는 ap-worker-1 (nodeSelector 효과)

# VM 콘솔 접속 (Ctrl+] 로 빠져나옴)
virtctl console test-vm-fedora -n vm-workload

# 콘솔 안에서 검증:
# ip addr
# eth0: 10.x.x.x (Pod network, masquerade)
# eth1: 192.168.20.150 (Service network)

# ping 192.168.20.1   (Service network gateway)
# ping 192.168.20.10  (Bastion Service NIC)
```

VM에서 Service망 통신이 가능하다면 NMState bridge + NAD 구성이 완전히 동작하는 것입니다.

> **virtctl 도구 설치 필요**
> `virtctl`은 OpenShift Virtualization의 CLI 도구입니다. Operator 설치 후 자동으로 클러스터 안에 다운로드 endpoint가 생기지만, 별도로 Bastion에 설치 필요:
>
> ```bash
> # 설치 (오프라인 환경)
> oc get consoleclidownload virtctl -o jsonpath='{.spec.links[?(@.text=="Download virtctl for Linux for x86_64")].href}'
> # 위 URL에서 다운로드 후 Bastion에 배치
> ```

### 5.7.3 검증 후 정리

```bash
oc delete vm test-vm-fedora -n vm-workload
```

---

## 5.8 NAS NFS StorageClass 구성

### 5.8.1 NAS 설계 (PoC)

Part 1.5.1에서 정의한 NAS:
- NAS VIP: 192.168.10.70
- Export 경로 (PoC 예시): `/exports/ocp-mtv`
- 용도: MTV 마이그레이션 대상 VM 디스크, 테스트 PVC

NAS 측 설정 (NAS 장비에서):
```
Export Path: /exports/ocp-mtv
Allowed Hosts: 192.168.10.0/24
Permissions: rw, no_root_squash, no_subtree_check
```

### 5.8.2 NFS Provisioner Operator 또는 직접 구성

OpenShift는 NFS용 dynamic provisioner를 기본 제공하지 않습니다. 두 가지 방법:

**옵션 A: NFS CSI Driver (권장)**

`csi-driver-nfs` Operator를 통해 dynamic provisioning 지원.

**옵션 B: Static PV 직접 생성 (단순)**

각 PV를 수동으로 생성. PoC 단계에서 간단히 시작.

본 PoC에서는 옵션 A를 다룹니다.

### 5.8.3 NFS CSI Driver 설치

NFS CSI Driver는 OperatorHub에 없을 수 있으므로 직접 매니페스트로 배포합니다.

```bash
mkdir -p ~/openshift-day2/nfs-csi
cd ~/openshift-day2/nfs-csi

# NFS CSI Driver 매니페스트 다운로드 (외부망에서 미리)
# https://github.com/kubernetes-csi/csi-driver-nfs

# PoC를 위한 간략화된 설치 (실제 매니페스트는 외부망에서 받아 옮김)
# 폐쇄망: 이미지 미러링 필요
#   registry.k8s.io/sig-storage/nfsplugin
#   registry.k8s.io/sig-storage/csi-provisioner
#   registry.k8s.io/sig-storage/csi-snapshotter
#   ...
```

**대안: 더 간단한 in-cluster NFS provisioner (Helm 기반)**

```bash
# helm 설치 (Bastion)
# (외부에서 받은 helm 바이너리 사용)

# nfs-subdir-external-provisioner chart 사용
# 이미지: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner
```

> **NFS Provisioner 선택은 환경 종속적**
> 폐쇄망에서 미러링 가능한 NFS provisioner를 선택해야 합니다. 본 가이드에서는 매니페스트 전체를 다루지 않고 개념만 설명하며, 실제 환경에서는 다음 중 하나를 사용:
> 1. **Red Hat 권장**: CSI를 지원하는 storage vendor의 driver (NetApp Trident, Pure Service Orchestrator 등)
> 2. **오픈소스**: `nfs-subdir-external-provisioner` (간단, 검증된)
> 3. **수동 PV**: PoC에서 빠르게 시작

### 5.8.4 옵션 B: 수동 PV로 시작 (가장 단순)

PoC에서는 미러링 부담을 피하기 위해 수동 PV로 시작할 수 있습니다.

```bash
cat > 09-nfs-pv.yaml <<'EOF'
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-mtv-pv-01
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-mtv
  nfs:
    path: /exports/ocp-mtv/pv01
    server: 192.168.10.70
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-mtv-pv-02
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-mtv
  nfs:
    path: /exports/ocp-mtv/pv02
    server: 192.168.10.70
EOF

oc apply -f 09-nfs-pv.yaml

# StorageClass (dummy, dynamic provisioning 안 함)
cat > 10-nfs-storageclass.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-mtv
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/no-provisioner   # static PV만 사용
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
EOF

oc apply -f 10-nfs-storageclass.yaml
```

> **수동 PV 방식의 한계**
> 미리 만든 PV 만큼만 PVC 요청을 받을 수 있습니다. MTV가 동시에 여러 VM을 마이그레이션하면 PV가 부족할 수 있습니다. NAS에 export를 추가하고 PV를 추가 생성하는 작업이 반복됩니다.
> 본격 운영 전에는 dynamic provisioning이 가능한 CSI driver 도입을 권장합니다.

### 5.8.5 NFS 동작 검증

```bash
cat > 11-test-pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs-pvc
  namespace: vm-workload
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs-mtv
EOF

oc apply -f 11-test-pvc.yaml

# 결과 확인
oc get pvc -n vm-workload
# 기대: test-nfs-pvc  Bound  nfs-mtv-pv-01 (또는 02)  100Gi  RWX

# PVC를 사용하는 Pod로 mount 검증
cat > 12-test-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test-pod
  namespace: vm-workload
spec:
  containers:
  - name: app
    image: mirror.ocp1.example.com:8443/ubi9/ubi:latest
    command: ["/bin/sh", "-c", "echo 'NFS works' > /data/test.txt; sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: nfs-data
  volumes:
  - name: nfs-data
    persistentVolumeClaim:
      claimName: test-nfs-pvc
EOF

oc apply -f 12-test-pod.yaml

# Pod이 Running되면 NAS에서 파일이 보이는지 확인
oc exec -n vm-workload nfs-test-pod -- cat /data/test.txt
# 기대: NFS works
```

검증 완료 후 정리:

```bash
oc delete -f 11-test-pvc.yaml -f 12-test-pod.yaml
```

---

## 5.9 IngressController 분리 구성

### 5.9.1 두 IngressController 배치 계획

Part 1.8에서 정한 설계를 실제 구현합니다.

| IngressController | 배치 | 도메인 | 외부 LB |
|---|---|---|---|
| `default` (관리용) | ap MCP | `apps.ocp1.example.com` | 192.168.10.21 (Bastion HAProxy Infra) |
| `service` (업무용) | podpool1 MCP | `svcapps.ocp1.example.com` | 192.168.20.20 (Bastion HAProxy Service) |

### 5.9.2 Default IngressController 수정 (ap로 이동)

설치 직후 default IngressController는 worker 전체에 배포되어 있습니다. ap MCP로 이동시킵니다.

```bash
cat > 13-ingress-default.yaml <<'EOF'
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  domain: apps.ocp1.example.com
  replicas: 2

  # ap MCP에 배치
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/ap: ""
    tolerations:
    - effect: NoSchedule
      key: workload
      operator: Equal
      value: vm

  # HostNetwork (80/443 점유)
  endpointPublishingStrategy:
    type: HostNetwork

  # 기본 정책: 모든 namespace의 Route 받음 (별도 selector 없음)
  # service IngressController가 라벨로 분리하므로 default는 default 동작 유지
EOF

oc apply -f 13-ingress-default.yaml
```

> **toleration의 의미**
> Part 4에서 ap MCP에 `workload=vm:NoSchedule` taint를 적용했다면, 그 taint를 router Pod가 견딜 수 있도록 toleration을 추가해야 합니다.
> 만약 ap MCP에 taint를 안 걸었다면 toleration도 불필요하지만, 명시적으로 둬도 부작용은 없습니다.

```bash
# router Pod 재배치 확인
watch 'oc get pods -n openshift-ingress -o wide'
```

기대:
```
NAME                              READY   STATUS    NODE
router-default-XXXXXXX            1/1     Running   ap-worker-0
router-default-XXXXXXX            1/1     Running   ap-worker-1
```

이전에 다른 노드에 있던 router Pod이 삭제되고 ap 노드에 새로 생성됩니다.

### 5.9.3 Service IngressController 신규 생성

```bash
cat > 14-ingress-service.yaml <<'EOF'
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: service
  namespace: openshift-ingress-operator
spec:
  domain: svcapps.ocp1.example.com
  replicas: 2

  # podpool1 MCP에 배치
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/podpool1: ""

  # HostNetwork (80/443 점유)
  endpointPublishingStrategy:
    type: HostNetwork

  # namespaceSelector로 분리
  # "ingress-type: service" 라벨이 있는 namespace의 Route만 admit
  namespaceSelector:
    matchLabels:
      ingress-type: service

  # 또는 routeSelector로 분리 (선택)
  # routeSelector:
  #   matchLabels:
  #     ingress-type: service
EOF

oc apply -f 14-ingress-service.yaml
```

### 5.9.4 router Pod 검증

```bash
oc get pods -n openshift-ingress -o wide
```

기대:
```
NAME                              NODE                    STATUS
router-default-XXXXXXX            ap-worker-0             Running
router-default-XXXXXXX            ap-worker-1             Running
router-service-XXXXXXX            podpool1-worker-0       Running
router-service-XXXXXXX            podpool1-worker-1       Running
```

총 4개의 router Pod이 의도한 노드에 배치되어야 합니다.

### 5.9.5 두 IngressController의 도메인 충돌 방지

본 PoC는 두 IngressController가 서로 다른 wildcard 도메인을 사용하므로 충돌하지 않습니다.

- default: `*.apps.ocp1.example.com`
- service: `*.svcapps.ocp1.example.com`

**자동 생성되는 관리 Route는 default IngressController가 admit:**

```bash
oc get route -n openshift-console
# console-openshift-console.apps.ocp1.example.com
```

**사용자가 만드는 업무 Route는 라벨링된 namespace에서만 service IngressController가 admit:**

```bash
# 업무 namespace에 라벨 부여
oc label namespace vm-workload ingress-type=service

# 그 namespace에서 만든 Route는 svcapps 도메인을 받음
```

### 5.9.6 통합 테스트

간단한 hello-app으로 두 IngressController를 검증합니다.

```bash
# 관리용 Route (default IngressController, ap 노드)
cat > 15-test-admin-route.yaml <<'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: admin-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-admin
  namespace: admin-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-admin
  template:
    metadata:
      labels:
        app: hello-admin
    spec:
      containers:
      - name: hello
        image: mirror.ocp1.example.com:8443/openshift/hello-openshift:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello-admin
  namespace: admin-test
spec:
  selector:
    app: hello-admin
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: hello-admin
  namespace: admin-test
spec:
  to:
    kind: Service
    name: hello-admin
EOF

oc apply -f 15-test-admin-route.yaml
```

```bash
# 업무용 Route (service IngressController, podpool1 노드)
cat > 16-test-service-route.yaml <<'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: service-test
  labels:
    ingress-type: service    # service IngressController가 admit
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-service
  namespace: service-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-service
  template:
    metadata:
      labels:
        app: hello-service
    spec:
      nodeSelector:
        node-role.kubernetes.io/podpool1: ""    # podpool1에 배치
      containers:
      - name: hello
        image: mirror.ocp1.example.com:8443/openshift/hello-openshift:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
  namespace: service-test
spec:
  selector:
    app: hello-service
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: hello-service
  namespace: service-test
spec:
  to:
    kind: Service
    name: hello-service
EOF

oc apply -f 16-test-service-route.yaml
```

```bash
# Route 도메인 확인
oc get route -A | grep hello
```

기대:
```
admin-test     hello-admin     hello-admin-admin-test.apps.ocp1.example.com         ...
service-test   hello-service   hello-service-service-test.svcapps.ocp1.example.com  ...
```

두 Route의 host가 각각 `apps`와 `svcapps`로 자동 분리됨을 확인할 수 있습니다.

```bash
# 접근 테스트 (Bastion에서)
curl -k https://hello-admin-admin-test.apps.ocp1.example.com
# Hello OpenShift!

curl -k https://hello-service-service-test.svcapps.ocp1.example.com
# Hello OpenShift!
```

두 응답이 모두 성공하면 IngressController 분리가 완전히 동작하는 것입니다.

검증 후 정리:

```bash
oc delete ns admin-test service-test
```

### 5.9.7 IngressController 분리 검증 매트릭스

| 검증 항목 | 확인 명령 | 기대 결과 |
|---|---|---|
| default router 배치 | `oc get pods -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default -o wide` | NODE = ap-worker-0/1 |
| service router 배치 | `oc get pods -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=service -o wide` | NODE = podpool1-worker-0/1 |
| default 도메인 | `oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}'` | apps.ocp1.example.com |
| service 도메인 | `oc get ingresscontroller service -n openshift-ingress-operator -o jsonpath='{.status.domain}'` | svcapps.ocp1.example.com |
| 관리 Route 동작 | `curl -k https://console-openshift-console.apps.ocp1.example.com` | 200 OK |
| 업무 Route 동작 | `curl -k https://test.svcapps.ocp1.example.com` | 적절한 응답 |

---

## 5.10 추가 Operator 설치

### 5.10.1 MTV Operator

```bash
cat > 17-mtv-operator.yaml <<'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-mtv
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: migration
  namespace: openshift-mtv
spec:
  targetNamespaces:
  - openshift-mtv
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: mtv-operator
  namespace: openshift-mtv
spec:
  channel: release-v2.7
  installPlanApproval: Automatic
  name: mtv-operator
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
EOF

oc apply -f 17-mtv-operator.yaml

# 상태
oc get csv -n openshift-mtv
```

ForkliftController CR 생성:

```bash
cat > 18-forklift-controller.yaml <<'EOF'
apiVersion: forklift.konveyor.io/v1beta1
kind: ForkliftController
metadata:
  name: forklift-controller
  namespace: openshift-mtv
spec:
  olm_managed: true
EOF

oc apply -f 18-forklift-controller.yaml

# MTV Pod 동작 확인
oc get pods -n openshift-mtv
```

세부 사용은 Part 6 (MTV 시나리오)에서 다룹니다.

### 5.10.2 OpenShift Pipelines (Tekton)

```bash
cat > 19-pipelines-operator.yaml <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-pipelines-operator-rh
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
EOF

oc apply -f 19-pipelines-operator.yaml
```

> **`openshift-operators` namespace 사용**
> 클러스터 전역 Operator는 `openshift-operators` namespace에 설치하는 것이 표준입니다. 이 namespace에는 기본 OperatorGroup이 있어 별도 정의 불필요합니다.

### 5.10.3 OpenShift GitOps

```bash
cat > 20-gitops-operator.yaml <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
EOF

oc apply -f 20-gitops-operator.yaml
```

### 5.10.4 Service Mesh

```bash
cat > 21-servicemesh-operator.yaml <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: servicemeshoperator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: servicemeshoperator
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
EOF

oc apply -f 21-servicemesh-operator.yaml
```

> **Service Mesh control plane**
> Operator 설치 후 `ServiceMeshControlPlane` CR을 별도로 생성해야 실제 control plane이 배포됩니다. 세부 구성은 Part 6에서 다룹니다.

### 5.10.5 Node Health Check + Fence Agents Remediation

```bash
cat > 22-nhc-far-operators.yaml <<'EOF'
---
# Node Health Check Operator
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-workload-availability
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: workload-availability
  namespace: openshift-workload-availability
spec:
  targetNamespaces:
  - openshift-workload-availability
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: node-healthcheck-operator
  namespace: openshift-workload-availability
spec:
  channel: stable
  installPlanApproval: Automatic
  name: node-healthcheck-operator
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
---
# Fence Agents Remediation Operator (같은 namespace)
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: fence-agents-remediation
  namespace: openshift-workload-availability
spec:
  channel: stable
  installPlanApproval: Automatic
  name: fence-agents-remediation
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
EOF

oc apply -f 22-nhc-far-operators.yaml

oc get csv -n openshift-workload-availability
```

세부 fencing 정책 구성은 Part 6에서 다룹니다.

### 5.10.6 모든 Operator 설치 상태 점검

```bash
# 모든 namespace의 CSV 확인
oc get csv -A | grep -v "system:" | grep -E "(NAME|Succeeded|Pending|Failed)"
```

기대 (모두 Succeeded):
```
NAMESPACE                       NAME                                       DISPLAY                            PHASE
openshift-cnv                   kubevirt-hyperconverged-operator.v4.20.0   OpenShift Virtualization           Succeeded
openshift-mtv                   mtv-operator.v2.7.0                        Migration Toolkit for Virt         Succeeded
openshift-nmstate               kubernetes-nmstate-operator.v4.20.0        Kubernetes NMState Operator        Succeeded
openshift-operators             openshift-gitops-operator.v1.x.x           Red Hat OpenShift GitOps           Succeeded
openshift-operators             openshift-pipelines-operator-rh.v1.x.x     Red Hat OpenShift Pipelines        Succeeded
openshift-operators             servicemeshoperator.v2.x.x                 Red Hat OpenShift Service Mesh     Succeeded
openshift-workload-availability node-healthcheck-operator.v0.x.x           Node Health Check Operator         Succeeded
openshift-workload-availability fence-agents-remediation.v0.x.x            Fence Agents Remediation           Succeeded
```

---

## 5.11 Part 5 종합 검증

### 5.11.1 인프라 컴포넌트 일괄 확인

```bash
# 1. 모든 노드 Ready
oc get nodes
# 9대 모두 Ready, 역할 라벨 부여 상태

# 2. 모든 MCP UPDATED
oc get mcp
# master, ap, db, podpool1 모두 UPDATED=True

# 3. NMState 동작
oc get nncp
# vm-service-bridge-ap, vm-service-bridge-db 모두 Available

# 4. Virtualization 동작
oc get hyperconverged -n openshift-cnv
# Available=True

# 5. virt-handler가 ap/db에만 배치
oc get pods -n openshift-cnv -l kubevirt.io=virt-handler -o wide
# 4개, NODE = ap/db 노드만

# 6. IngressController 분리
oc get ingresscontroller -n openshift-ingress-operator
# default, service 두 개

oc get pods -n openshift-ingress -o wide
# default 2개 (ap), service 2개 (podpool1)

# 7. NAS PV 동작
oc get pv
# 100Gi PV들 Available

# 8. 모든 Operator Succeeded
oc get csv -A | grep -v "Succeeded\|NAME"
# 비어있어야 함 (모두 Succeeded)
```

### 5.11.2 네트워크 흐름 점검

다음 흐름이 모두 동작해야 합니다.

```
[관리 사이클 - 컨테이너 트래픽]
관리자 브라우저
   ↓ https://console-openshift-console.apps.ocp1.example.com
DNS → 192.168.10.21 (Bastion HAProxy Default Ingress VIP)
   ↓
HAProxy → ap-worker-0/1:443 (default router HostNetwork)
   ↓
default IngressController → openshift-console namespace
   ↓
console Pod 응답
```

```
[서비스 사이클 - 컨테이너 트래픽]
사용자 브라우저
   ↓ https://app.svcapps.ocp1.example.com
DNS → 192.168.20.20 (Bastion HAProxy Service Ingress VIP)
   ↓
HAProxy → podpool1-worker-0/1:443 (service router HostNetwork)
   ↓
service IngressController → 업무 namespace
   ↓
업무 Pod 응답
```

```
[VM 사이클 - VM 트래픽 (Bastion HAProxy 미경유)]

▶ Outbound (VM → 외부)
VM Guest OS (192.168.20.150)
   ↓ Guest OS의 default gateway: 192.168.20.1
NAD (vm-service-net, cnv-bridge)
   ↓
Linux bridge: br-vm-svc (호스트 IP 없음, L2 통로)
   ↓
ens224 (host NIC, IP 없음, bridge port)
   ↓
Service망 스위치 (192.168.20.0/24)
   ↓
외부 시스템

▶ Inbound (외부 → VM)
외부 사용자
   ↓ VM Guest IP(192.168.20.150)로 직접 접근
     또는 DNS 등록된 VM hostname
Service망 스위치 (192.168.20.0/24)
   ↓
ap/db 노드의 ens224 (bridge port)
   ↓
br-vm-svc → NAD → VM Guest OS

★ 어느 단계에서도 Bastion HAProxy를 거치지 않음
★ Bastion Service Ingress VIP(20.20)는 컨테이너 Route 전용
```

---

## 5.12 Part 5 학습 점검

다음 질문에 답할 수 있다면 Part 5를 충분히 학습한 것입니다.

1. OLM의 핵심 리소스 6가지(CatalogSource, PackageManifest, OperatorGroup, Subscription, InstallPlan, CSV)는 각각 어떤 역할을 하는가?
2. 본 PoC가 폐쇄망에서 외부 OperatorHub를 비활성화하는 이유와 방법은?
3. NMState Operator를 OpenShift Virtualization보다 먼저 설치해야 하는 이유는?
4. NNCP의 `bridge`에 호스트 IP를 부여하면 안 되는 이유는?
5. NNCP 적용 중 노드 네트워크가 끊겨도 안전한 이유는?
6. HyperConverged CR의 `infra.nodePlacement`와 `workloads.nodePlacement`의 차이는?
7. virt-handler가 podpool1 노드에 배치되지 않아야 하는 이유는?
8. NetworkAttachmentDefinition의 `cnv-bridge` 타입이 일반 `bridge`와 다른 점은?
9. NAD에서 IPAM을 사용하지 않는 결정의 trade-off는?
10. PoC에서 NFS dynamic provisioning 대신 수동 PV로 시작하는 이유와 한계는?
11. Default IngressController를 ap MCP로 이동시킬 때 toleration이 필요한 이유는?
12. Service IngressController에 `namespaceSelector: ingress-type=service`를 설정하면 어떤 효과가 있는가?
13. 두 IngressController가 같은 노드(예: podpool1)에 함께 배포되면 어떤 문제가 발생하는가?

---

*Part 5 끝. 다음은 Part 6 (MTV 마이그레이션, FAR/vBMC fencing, 트러블슈팅) 입니다.*
