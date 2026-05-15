# Part 6. 시나리오 테스트와 운영 통합

> **이 파트의 목적**
> Part 5까지 구축한 플랫폼 위에서 본 PoC의 핵심 시나리오를 검증합니다.
>
> 1. **MTV**: 외부 가상화 플랫폼(VMware vSphere 등)의 VM을 OpenShift Virtualization으로 마이그레이션
> 2. **FAR + vBMC**: 노드 장애 발생 시 자동 fencing 동작 검증
> 3. **업무 워크로드 배치**: Tekton, GitOps, Service Mesh, JBoss를 podpool1에 배치
> 4. **업무 Egress 검증**: EgressIP를 통한 Service망 outbound 제어 PoC
> 5. **종합 검증과 트러블슈팅**: 전체 클러스터의 헬스 점검과 흔한 문제 해결
> 6. **PoC → 프로덕션 전환 체크리스트**: 변경 항목 일괄 정리
>
> 이 파트를 학습하고 나면 PoC 환경에서 의미 있는 시나리오를 직접 설계·실행할 수 있고, 운영 환경 전환 시점에 무엇을 점검해야 하는지 명확히 알 수 있어야 합니다.

---

## 6.1 MTV (Migration Toolkit for Virtualization) 시나리오

### 6.1.1 MTV 개요

**MTV**는 외부 가상화 플랫폼의 VM을 OpenShift Virtualization으로 마이그레이션하는 도구입니다. Konveyor Forklift 프로젝트 기반으로, Red Hat이 상용 지원합니다.

지원 source provider:

| Provider | 설명 | 본 PoC 권장 |
|---|---|---|
| VMware vSphere | 가장 일반적, ESXi/vCenter 기반 | 검증 권장 |
| Red Hat Virtualization (RHV) | KVM 기반 가상화 | 환경 있다면 검증 |
| OpenStack | 클라우드 플랫폼 | 환경 있다면 |
| OVA file | OVA 형식 단일 VM 이미지 | 가장 단순한 검증용 |
| OpenShift Virtualization | OCP-V 간 마이그레이션 | |

### 6.1.2 MTV 아키텍처

```
[Source]                          [OpenShift]
vSphere/RHV                        Target Provider
   │                                  │
   │                                  ▼
   │                            HyperConverged
   │                            (OpenShift Virtualization)
   │                                  ▲
   │                                  │
   │            Plan ─────────────────┤
   │             │                    │
   │             ├─ Network Mapping   │
   │             ├─ Storage Mapping   │
   │             └─ VM Selection      │
   ▼                                  │
Source Provider                       │
   │                                  │
   └─── MTV Engine (forklift) ────────┘
        │
        ├─ virt-v2v (디스크 변환)
        ├─ CDI importer (PVC에 디스크 import)
        └─ KubeVirt VM 생성
```

### 6.1.3 MTV CR 계층

| CR | 역할 |
|---|---|
| `Provider` | Source 또는 Target 환경 정의 |
| `NetworkMap` | Source 네트워크 ↔ NAD 매핑 |
| `StorageMap` | Source 스토리지 ↔ OpenShift StorageClass 매핑 |
| `Plan` | 마이그레이션 계획 (어느 VM을 옮길지, 어떤 Map을 쓸지) |
| `Migration` | Plan 실행 인스턴스 (실제 마이그레이션 작업) |

### 6.1.4 vSphere Source Provider 등록

가장 일반적인 시나리오를 가정하고 vSphere를 source로 등록합니다.

**vCenter 자격증명 Secret:**

```bash
mkdir -p ~/openshift-day2/mtv
cd ~/openshift-day2/mtv

# Secret 생성 (vCenter 자격증명 + thumbprint)
oc create secret generic vsphere-secret \
  --from-literal=user='administrator@vsphere.local' \
  --from-literal=password='vCenterPassword!' \
  --from-literal=thumbprint='AA:BB:CC:DD:...' \
  -n openshift-mtv
```

> **thumbprint 확인 방법**
> vCenter의 SSL 인증서 SHA-1 thumbprint입니다. 다음 명령으로 추출 가능:
> ```bash
> openssl s_client -connect vcenter.example.com:443 < /dev/null 2>/dev/null | \
>   openssl x509 -noout -fingerprint -sha1 | \
>   awk -F= '{print $2}'
> # AA:BB:CC:DD:...
> ```

**Provider CR:**

```yaml
cat > 23-mtv-vsphere-provider.yaml <<'EOF'
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata:
  name: vsphere-source
  namespace: openshift-mtv
spec:
  type: vsphere
  url: 'https://vcenter.example.com/sdk'
  secret:
    name: vsphere-secret
    namespace: openshift-mtv
  settings:
    sdkEndpoint: vcenter
EOF

oc apply -f 23-mtv-vsphere-provider.yaml

# 상태 확인
oc get provider -n openshift-mtv
```

기대 (수 분 후):
```
NAME              TYPE       STATUS   URL
host              openshift  Ready    (default OCP target)
vsphere-source    vsphere    Ready    https://vcenter.example.com/sdk
```

> **Target Provider는 자동 생성**
> MTV는 설치 시점에 자기 클러스터(OpenShift Virtualization)를 target provider로 자동 등록합니다. 별도 등록 작업이 필요 없습니다.

### 6.1.5 Network Mapping

vSphere의 네트워크와 OpenShift의 NAD를 매핑합니다.

```yaml
cat > 24-mtv-network-map.yaml <<'EOF'
apiVersion: forklift.konveyor.io/v1beta1
kind: NetworkMap
metadata:
  name: vsphere-network-map
  namespace: openshift-mtv
spec:
  provider:
    source:
      name: vsphere-source
      namespace: openshift-mtv
    destination:
      name: host
      namespace: openshift-mtv
  map:
  # vSphere "VM Network" → OpenShift NAD vm-service-net
  - source:
      name: VM Network        # vSphere에서의 portgroup 이름
      type: standard
    destination:
      name: vm-service-net    # Part 5.6에서 만든 NAD
      namespace: vm-workload
      type: multus

  # 또는 Pod network로 매핑 (격리된 VM)
  - source:
      name: Internal Network
      type: standard
    destination:
      type: pod
EOF

oc apply -f 24-mtv-network-map.yaml

oc get networkmap -n openshift-mtv
```

> **vSphere에서 사용 가능한 네트워크 이름 조회**
> ```bash
> # MTV Provider가 vSphere에 연결되어 있으면 inventory API로 조회 가능
> oc get provider vsphere-source -n openshift-mtv -o jsonpath='{.status.conditions}'
> # 또는 vCenter UI에서 직접 확인
> ```

### 6.1.6 Storage Mapping

vSphere의 datastore와 OpenShift의 StorageClass를 매핑합니다.

```yaml
cat > 25-mtv-storage-map.yaml <<'EOF'
apiVersion: forklift.konveyor.io/v1beta1
kind: StorageMap
metadata:
  name: vsphere-storage-map
  namespace: openshift-mtv
spec:
  provider:
    source:
      name: vsphere-source
      namespace: openshift-mtv
    destination:
      name: host
      namespace: openshift-mtv
  map:
  # vSphere datastore → OpenShift StorageClass
  - source:
      name: datastore1      # vSphere datastore 이름
    destination:
      storageClass: nfs-mtv  # Part 5.8에서 만든 StorageClass
      accessMode: ReadWriteMany
      volumeMode: Filesystem
EOF

oc apply -f 25-mtv-storage-map.yaml
```

### 6.1.7 Migration Plan 작성

어떤 VM을 어떻게 마이그레이션할지 정의합니다.

```yaml
cat > 26-mtv-plan.yaml <<'EOF'
apiVersion: forklift.konveyor.io/v1beta1
kind: Plan
metadata:
  name: migrate-test-vms
  namespace: openshift-mtv
spec:
  provider:
    source:
      name: vsphere-source
      namespace: openshift-mtv
    destination:
      name: host
      namespace: openshift-mtv

  # 마이그레이션된 VM이 들어갈 namespace
  targetNamespace: vm-workload

  # Map 참조
  map:
    network:
      name: vsphere-network-map
      namespace: openshift-mtv
    storage:
      name: vsphere-storage-map
      namespace: openshift-mtv

  # 마이그레이션 모드
  # warm: source가 켜져있는 동안 디스크 복사 (precopy), cutover 시점에 잠깐 중단
  # cold: source를 끄고 디스크 전체 복사 (다운타임 큼, 단순)
  warm: true

  # 마이그레이션할 VM 목록
  vms:
  - name: test-app-vm-01
    namespace: openshift-mtv
  - name: test-db-vm-01
    namespace: openshift-mtv
EOF

oc apply -f 26-mtv-plan.yaml

# Plan 상태 확인
oc get plan -n openshift-mtv
```

기대 (검증 후):
```
NAME                READY   PENDING   RUNNING   SUCCEEDED   FAILED
migrate-test-vms    True    0         0         0           0
```

`Ready=True`이면 Plan이 검증되어 실행 가능 상태입니다.

### 6.1.8 마이그레이션 실행

Plan을 만들었어도 자동 실행되지 않습니다. Migration CR로 명시적으로 실행해야 합니다.

```yaml
cat > 27-mtv-migration.yaml <<'EOF'
apiVersion: forklift.konveyor.io/v1beta1
kind: Migration
metadata:
  name: migrate-test-vms-run1
  namespace: openshift-mtv
spec:
  plan:
    name: migrate-test-vms
    namespace: openshift-mtv

  # cutover 시점 (warm 모드 전용)
  # 명시 안 하면 즉시 cutover
  # cutover: '2026-05-15T22:00:00Z'
EOF

oc apply -f 27-mtv-migration.yaml

# 진행 모니터링
watch 'oc get migration -n openshift-mtv; echo "---"; oc get plan -n openshift-mtv'
```

기대 단계별 진행:

1. **Initialization**: Plan 검증, target namespace 준비
2. **DiskTransfer (warm 모드)**: precopy 시작, source VM은 계속 실행
3. **DiskTransfer 반복**: 변경분 추적 및 동기화
4. **Cutover**: source VM 중단 → 최종 디스크 동기화 → OpenShift VM 생성
5. **VMCreation**: VirtualMachine CR 생성
6. **Completed**: 완료

```bash
# 마이그레이션된 VM 확인
oc get vm -n vm-workload
oc get vmi -n vm-workload -o wide

# VM이 ap 또는 db 노드에 배치되었는지 확인 (Part 5의 nodePlacement 적용)
```

### 6.1.9 마이그레이션 후 검증

```bash
# VM 콘솔 접속
virtctl console test-app-vm-01 -n vm-workload

# VM 내부에서:
# 1. 네트워크 확인 - Service망 통신 가능한지
ping 192.168.20.1
ping 192.168.20.10  # Bastion

# 2. 기존 VM의 서비스가 동작하는지
systemctl status <원래 서비스>

# 3. 디스크 무결성
df -h
```

### 6.1.10 MTV 시나리오 흔한 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| Provider가 NotReady, 인증 실패 | vCenter 자격증명 또는 thumbprint 오류 | Secret 재생성 |
| Plan이 Ready=False | NetworkMap 또는 StorageMap 매칭 실패 | vSphere의 네트워크/datastore 이름 정확히 확인 |
| Migration이 DiskTransfer에서 멈춤 | PVC 생성 실패 (NAS PV 부족) | PV 추가 생성 또는 동적 provisioner 사용 |
| VM이 부팅 후 네트워크 안 됨 | VM Guest OS의 NIC 이름 변경 (eth0 → ens3) | Guest 내부에서 NIC 설정 수정, 또는 NAD MAC 정책 검토 |
| virt-v2v 변환 실패 | source VM의 OS가 지원되지 않음 | MTV 호환성 매트릭스 확인 (RHEL, Windows 등) |

> **PoC에서 외부 vSphere가 없다면**
> OVA 파일 import 시나리오로 대체 가능합니다. Provider type을 `ova`로 등록하고, OVA 파일을 HTTP 서버에 두면 됩니다. 또는 OpenShift Virtualization에 미리 만든 VM을 그대로 사용해도 마이그레이션 자체는 검증 안 되지만 nodePlacement/NAD 동작은 확인 가능합니다.

---

## 6.2 FAR + vBMC Fencing 시나리오

### 6.2.1 Fencing의 필요성

**Fencing**은 응답 없는 노드를 강제로 격리(전원 차단 또는 재부팅)하는 작업입니다. Kubernetes는 노드 상태가 NotReady이면 그 노드의 Pod을 다른 노드로 옮기려 하지만, 일부 워크로드는 **fencing 없이는 안전하게 옮길 수 없습니다.**

| 워크로드 | Fencing 필요 이유 |
|---|---|
| StatefulSet (DB) | 같은 PVC를 두 Pod이 쓰면 데이터 corruption |
| VM (KubeVirt) | 같은 디스크를 두 hypervisor가 쓰면 디스크 손상 |
| RWO PVC 사용 Pod | 노드 간 강제 detach 시 데이터 손실 |

Fencing이 적용되면:
1. NotReady 노드 감지
2. 외부 인터페이스(BMC, IPMI)로 노드 전원 차단
3. 전원 차단 확인 후 Pod taint 제거
4. Scheduler가 다른 노드에 Pod 재배치

### 6.2.2 NHC + FAR 흐름

| 컴포넌트 | 역할 |
|---|---|
| **Node Health Check (NHC)** | 노드 상태 모니터링, 비정상 노드 감지 |
| **Fence Agents Remediation (FAR)** | NHC가 호출, 실제 fencing 수행 (IPMI/BMC) |
| **FenceAgentsRemediationTemplate** | FAR가 사용할 fence agent와 자격증명 정의 |
| **vBMC** | PoC 환경에서 VM에 BMC를 에뮬레이션 |

```
[NHC]
   ↓ (60초간 NotReady 감지)
"노드가 비정상"
   ↓
[FAR]
   ↓ FenceAgentsRemediationTemplate 참조
"fence_ipmilan으로 전원 차단"
   ↓
[vBMC]
   ↓ (IPMI 명령 수신)
"libvirt API로 VM 전원 OFF"
   ↓
[Hypervisor (libvirt)]
   ↓
VM 종료
```

### 6.2.3 vBMC 환경 준비

vBMC는 가상 머신을 위한 BMC(Baseboard Management Controller)를 에뮬레이션하는 도구입니다. Part 2에서 Bastion에 패키지를 설치만 해두었습니다. 이제 동작 환경을 구성합니다.

**Bastion에서 vBMC 설치 (재확인):**

```bash
# Bastion에서
pip3 install virtualbmc

# 데몬 시작
vbmcd

# 자동 시작
cat > /etc/systemd/system/vbmcd.service <<'EOF'
[Unit]
Description=VirtualBMC
After=libvirtd.service

[Service]
Type=forking
User=root
ExecStart=/usr/local/bin/vbmcd
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now vbmcd
```

> **vBMC가 libvirt와 같은 호스트에 있어야 함**
> vBMC는 libvirt API를 통해 VM 전원을 제어합니다. 따라서 OpenShift 노드 VM이 동작하는 hypervisor 호스트에서 vBMC가 실행되어야 합니다.
> PoC 환경에 따라:
> - **libvirt 직접**: hypervisor가 KVM이면 vBMC를 그 호스트에 설치
> - **VMware**: vBMC 대신 fence_vmware_rest 사용 (vCenter API로 VM 제어)
> - **별도 BMC 에뮬레이터 호스트**: vBMC 전용 호스트를 만들고 libvirt URL을 원격으로 설정
>
> 본 PoC는 KVM 환경을 가정하며, vBMC가 Bastion에서 동작한다고 가정합니다.

### 6.2.4 vBMC에 노드 등록

각 OpenShift 노드 VM에 BMC endpoint를 매핑합니다.

```bash
# Bastion에서

# 노드별 vBMC 인스턴스 생성
# 형식: vbmc add <VM-name> --port <port> --username admin --password password [--libvirt-uri qemu:///system]

vbmc add ap-worker-0 --port 6230 --username admin --password 'fenceP@ss!' --libvirt-uri qemu:///system
vbmc add ap-worker-1 --port 6231 --username admin --password 'fenceP@ss!' --libvirt-uri qemu:///system
vbmc add db-worker-0 --port 6232 --username admin --password 'fenceP@ss!' --libvirt-uri qemu:///system
vbmc add db-worker-1 --port 6233 --username admin --password 'fenceP@ss!' --libvirt-uri qemu:///system
vbmc add podpool1-worker-0 --port 6234 --username admin --password 'fenceP@ss!' --libvirt-uri qemu:///system
vbmc add podpool1-worker-1 --port 6235 --username admin --password 'fenceP@ss!' --libvirt-uri qemu:///system

# 각 vBMC 인스턴스 시작
for p in 6230 6231 6232 6233 6234 6235; do
  vbmc start $(vbmc list -f value -c "Domain name" | head -$((p - 6229)) | tail -1)
done

# 또는 일괄
for vm in ap-worker-0 ap-worker-1 db-worker-0 db-worker-1 podpool1-worker-0 podpool1-worker-1; do
  vbmc start $vm
done

# 등록 확인
vbmc list
```

기대:
```
+-------------------+----------+---------+------+
| Domain name       | Status   | Address | Port |
+-------------------+----------+---------+------+
| ap-worker-0       | running  | ::      | 6230 |
| ap-worker-1       | running  | ::      | 6231 |
| db-worker-0       | running  | ::      | 6232 |
| db-worker-1       | running  | ::      | 6233 |
| podpool1-worker-0 | running  | ::      | 6234 |
| podpool1-worker-1 | running  | ::      | 6235 |
+-------------------+----------+---------+------+
```

### 6.2.5 vBMC 동작 검증 (ipmitool)

```bash
# Bastion에서 ipmitool로 vBMC 호출
ipmitool -I lanplus -H 127.0.0.1 -p 6230 -U admin -P 'fenceP@ss!' chassis power status
# 기대: Chassis Power is on

# 다른 노드 (db-worker-0)
ipmitool -I lanplus -H 127.0.0.1 -p 6232 -U admin -P 'fenceP@ss!' chassis power status
```

이 명령이 동작하면 vBMC가 정상 동작하는 것입니다.

### 6.2.6 FenceAgentsRemediationTemplate 작성

FAR이 사용할 fence agent와 자격증명을 정의합니다.

```bash
mkdir -p ~/openshift-day2/fencing
cd ~/openshift-day2/fencing

# 노드별 BMC 정보를 담은 ConfigMap
cat > 28-fence-bmc-config.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: fence-bmc-info
  namespace: openshift-workload-availability
data:
  # 각 노드별 BMC 매핑 (참고용, 실제 FAR CR에서 사용)
  ap-worker-0: "192.168.10.10:6230"
  ap-worker-1: "192.168.10.10:6231"
  db-worker-0: "192.168.10.10:6232"
  db-worker-1: "192.168.10.10:6233"
  podpool1-worker-0: "192.168.10.10:6234"
  podpool1-worker-1: "192.168.10.10:6235"
EOF

oc apply -f 28-fence-bmc-config.yaml
```

```bash
# FAR Template (각 노드별 별도 또는 공통)
cat > 29-fence-template.yaml <<'EOF'
apiVersion: fence-agents-remediation.medik8s.io/v1alpha1
kind: FenceAgentsRemediationTemplate
metadata:
  name: fence-ipmilan-template
  namespace: openshift-workload-availability
spec:
  template:
    spec:
      # 사용할 fence agent
      agent: fence_ipmilan

      # Agent에 전달할 공통 파라미터
      sharedparameters:
        '--ip': '192.168.10.10'        # vBMC IP (Bastion)
        '--username': 'admin'
        '--password': 'fenceP@ss!'
        '--lanplus': ''
        '--action': 'reboot'           # reboot = power off + on

      # 노드별 파라미터 (포트 번호로 식별)
      nodeparameters:
        '--ipport':
          ap-worker-0: '6230'
          ap-worker-1: '6231'
          db-worker-0: '6232'
          db-worker-1: '6233'
          podpool1-worker-0: '6234'
          podpool1-worker-1: '6235'

      # remediation 정책
      remediationStrategy: ResourceDeletion
EOF

oc apply -f 29-fence-template.yaml
```

> **`remediationStrategy` 옵션**
> - `ResourceDeletion`: fencing 후 노드의 모든 Pod, VolumeAttachment 삭제 (빠른 재배치)
> - `OutOfServiceTaint`: 노드에 out-of-service taint 적용 (KubeVirt VM에 적합)
>
> VM 워크로드가 있는 경우 `OutOfServiceTaint`를 권장하지만, 본 PoC는 단순성을 위해 `ResourceDeletion` 사용.

### 6.2.7 NodeHealthCheck CR 작성

NHC가 어떤 노드를 모니터링하고, 비정상 시 어떤 FAR Template을 호출할지 정의합니다.

```bash
cat > 30-nhc.yaml <<'EOF'
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: nhc-all-workers
spec:
  # 모니터링 대상 노드 (worker 라벨이 있는 모든 노드)
  selector:
    matchExpressions:
    - key: node-role.kubernetes.io/worker
      operator: Exists

  # 비정상 판정 기준
  unhealthyConditions:
  - type: Ready
    status: "False"
    duration: 60s    # 60초간 Ready=False면 비정상
  - type: Ready
    status: Unknown
    duration: 60s

  # 동시 remediation 제한 (전체의 49% 또는 명시 개수)
  minHealthy: 51%    # 51% 이상 healthy 유지

  # 호출할 remediation
  remediationTemplate:
    apiVersion: fence-agents-remediation.medik8s.io/v1alpha1
    kind: FenceAgentsRemediationTemplate
    name: fence-ipmilan-template
    namespace: openshift-workload-availability
EOF

oc apply -f 30-nhc.yaml

# 상태 확인
oc get nhc
```

기대:
```
NAME              ENABLED   STATUS
nhc-all-workers   true      observed 6 healthy nodes (master는 제외)
```

> **master는 NHC에서 제외**
> 본 PoC의 NHC는 worker만 대상으로 합니다. master fencing은 etcd quorum 등 복잡한 의존성이 있어 별도 정책이 필요하며, 일반적으로 사람의 판단을 거치는 것이 안전합니다.

### 6.2.8 Fencing 시나리오 테스트

실제 fencing 동작을 검증합니다.

**시나리오: podpool1-worker-1 강제 정지**

```bash
# 1. 테스트 대상 노드 확인
oc get nodes podpool1-worker-1

# 2. 노드 위의 워크로드 확인 (Pod 재배치를 관찰하기 위함)
oc get pods -A -o wide --field-selector spec.nodeName=podpool1-worker-1

# 3. 노드 VM을 강제 정지 (hypervisor에서 직접)
# Bastion에서:
virsh destroy podpool1-worker-1
# 또는 vSphere/관리 콘솔에서 power off

# 4. NHC가 감지하는 60초 대기
watch 'oc get nodes podpool1-worker-1; echo "---"; oc get fenceagentsremediation -A'
```

진행 기대:
```
NAME                   STATUS     ROLES
podpool1-worker-1      NotReady   podpool1,worker

# 60초 후
NAME                                           AGE    STATUS
fenceagentsremediation/podpool1-worker-1       30s    Pending → InProgress → Succeeded
```

```bash
# 5. FAR 동작 로그
oc logs -n openshift-workload-availability -l app.kubernetes.io/component=fence-agents-remediation-controller

# 6. vBMC 로그 (Bastion)
journalctl -u vbmcd -f
# 기대: IPMI 명령 수신 → libvirt API 호출

# 7. VM 상태 확인
virsh list --all | grep podpool1-worker-1
# 기대: shut off → (FAR가 reboot 명령이라면) running으로 복귀
```

**기대 흐름:**
1. 노드 강제 정지 (수동)
2. NHC가 NotReady 60초 감지
3. FAR가 IPMI reboot 명령 호출
4. vBMC가 libvirt를 통해 VM 재부팅
5. VM이 다시 부팅되어 OpenShift에 재가입
6. 노드 상태가 다시 Ready

```bash
# 8. Pod 재배치 확인
oc get pods -A -o wide --field-selector spec.nodeName=podpool1-worker-1
# 기대: NotReady 시점의 Pod들이 다른 노드로 옮겨졌어야 함
```

### 6.2.9 Fencing 흔한 문제

| 증상 | 원인 | 해결 |
|---|---|---|
| FAR Pod이 시작 안 됨 | NHC가 트리거하지 않음 | NHC selector 매칭 확인 |
| `fence_ipmilan` 실행 시 connection refused | vBMC 미실행 또는 방화벽 차단 | `vbmc list`, `ipmitool` 수동 테스트 |
| 노드는 fencing되었으나 Pod 재배치 안 됨 | RWO PVC의 VolumeAttachment 잔존 | `OutOfServiceTaint` 전략으로 변경 |
| Fencing 후 노드가 자동 복귀 안 됨 | `--action=off`로 설정됨 | `--action=reboot`으로 변경 |
| 정상 노드도 자꾸 fencing됨 | NHC 임계치가 너무 짧음 (60초 → 일시적 지연도 fencing) | `unhealthyConditions.duration`을 5분 이상으로 |

> **PoC에서 fencing 테스트는 신중하게**
> 실수로 너무 짧은 임계치를 두면 일시적 네트워크 hiccup으로도 노드가 재부팅되어 PoC가 불안정해질 수 있습니다. PoC 검증 시에만 NHC를 enable하고, 평소에는 `spec.suspend: true`로 비활성화하는 운영도 고려해보세요.

---

## 6.3 업무 워크로드 배치

### 6.3.1 OpenShift Pipelines (Tekton) 검증

Operator는 Part 5.10.2에서 설치했습니다. 간단한 Pipeline을 만들어 podpool1에 동작하는지 검증합니다.

```bash
mkdir -p ~/openshift-day2/workloads
cd ~/openshift-day2/workloads
```

```bash
cat > 31-tekton-test.yaml <<'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: tekton-test
  labels:
    ingress-type: service
  annotations:
    openshift.io/node-selector: 'node-role.kubernetes.io/podpool1='
---
# 간단한 Task
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: hello
  namespace: tekton-test
spec:
  steps:
  - name: echo
    image: mirror.ocp1.example.com:8443/ubi9/ubi:latest
    script: |
      #!/bin/bash
      echo "Hello from Tekton on podpool1!"
      hostname
      cat /etc/os-release
---
# Task 실행
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  name: hello-run
  namespace: tekton-test
spec:
  taskRef:
    name: hello
EOF

oc apply -f 31-tekton-test.yaml

# 실행 확인
oc get taskrun -n tekton-test
oc logs -n tekton-test -l tekton.dev/taskRun=hello-run -c step-echo
```

기대 출력:
```
Hello from Tekton on podpool1!
hello-run-pod
NAME="Red Hat Enterprise Linux"
```

> **`openshift.io/node-selector` 어노테이션의 역할**
> namespace에 이 어노테이션을 두면 그 namespace의 모든 Pod이 자동으로 해당 nodeSelector를 갖습니다. 사용자가 매번 nodeSelector를 명시하지 않아도 됩니다. 본 PoC에서 업무 namespace에 이 어노테이션을 일괄 적용하면 podpool1 격리가 자동으로 보장됩니다.

### 6.3.2 OpenShift GitOps (ArgoCD) 검증

Operator는 Part 5.10.3에서 설치했습니다. ArgoCD 인스턴스가 자동으로 `openshift-gitops` namespace에 생성됩니다.

```bash
# GitOps namespace의 Pod 위치 확인
oc get pods -n openshift-gitops -o wide
```

기본 상태에서는 worker 전체에 배치됩니다. podpool1 격리를 위해 `ArgoCD` CR을 수정합니다.

```bash
oc get argocd openshift-gitops -n openshift-gitops -o yaml > argocd-current.yaml

# nodeSelector 추가
oc patch argocd openshift-gitops -n openshift-gitops --type=merge -p '
spec:
  nodePlacement:
    nodeSelector:
      node-role.kubernetes.io/podpool1: ""
'

# 적용 확인 (Pod 재배치 발생)
watch 'oc get pods -n openshift-gitops -o wide'
```

### 6.3.3 Service Mesh 검증

Operator는 Part 5.10.4에서 설치했습니다. Service Mesh control plane을 별도 CR로 생성합니다.

```bash
cat > 32-servicemesh-cp.yaml <<'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: istio-system
---
apiVersion: maistra.io/v2
kind: ServiceMeshControlPlane
metadata:
  name: basic
  namespace: istio-system
spec:
  version: v2.6
  tracing:
    type: None    # PoC 단순화
  policy:
    type: Istiod
  telemetry:
    type: Istiod
  addons:
    grafana:
      enabled: false
    jaeger:
      install:
        storage:
          type: Memory
    kiali:
      enabled: true
    prometheus:
      enabled: true

  # podpool1에 배치
  runtime:
    defaults:
      pod:
        nodeSelector:
          node-role.kubernetes.io/podpool1: ""
EOF

oc apply -f 32-servicemesh-cp.yaml

# 진행 확인 (수 분 소요)
oc get smcp -n istio-system
```

기대:
```
NAME    READY   STATUS            PROFILES   VERSION
basic   9/9     ComponentsReady   ["default"]   2.6.x
```

### 6.3.4 JBoss EAP 배포 검증

Part 3에서 미러링한 JBoss EAP 8 base image를 사용한 간단한 배포.

```bash
cat > 33-jboss-eap.yaml <<'EOF'
---
apiVersion: v1
kind: Namespace
metadata:
  name: jboss-test
  labels:
    ingress-type: service
  annotations:
    openshift.io/node-selector: 'node-role.kubernetes.io/podpool1='
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eap-app
  namespace: jboss-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: eap-app
  template:
    metadata:
      labels:
        app: eap-app
    spec:
      containers:
      - name: eap
        image: mirror.ocp1.example.com:8443/jboss-eap-8/eap8-openjdk17-runtime-openshift-rhel8:latest
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1
            memory: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: eap-app
  namespace: jboss-test
spec:
  selector:
    app: eap-app
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: eap-app
  namespace: jboss-test
spec:
  to:
    kind: Service
    name: eap-app
  port:
    targetPort: 8080
  tls:
    termination: edge
EOF

oc apply -f 33-jboss-eap.yaml

# 결과 확인
oc get pods -n jboss-test -o wide
# 기대: podpool1 노드에 배치

oc get route -n jboss-test
# 기대: eap-app-jboss-test.svcapps.ocp1.example.com  (service IngressController)

# 접근 테스트
curl -k https://$(oc get route eap-app -n jboss-test -o jsonpath='{.spec.host}')
```

기대: JBoss EAP의 환영 페이지 또는 health check 응답.

> **실제 애플리케이션 빌드는 별도**
> 위 예시는 EAP base image를 그대로 배포한 것입니다. 실제 운영에서는 Source-to-Image (S2I) 또는 Pipeline을 통해 사용자 애플리케이션 WAR/JAR을 EAP base에 빌드하여 배포합니다. 이 과정은 본 PoC 범위를 벗어나는 별도 시나리오입니다.

---

## 6.4 EgressIP 검증 (업무 Egress)

### 6.4.1 EgressIP의 목적

Part 1.9.3에서 정의한 대로, 업무 Pod의 outbound를 Service망(192.168.20.0/24)으로 강제하기 위한 메커니즘을 PoC로 검증합니다.

**검증 목표:**
- `jboss-test` namespace의 Pod이 외부로 나갈 때 source IP가 192.168.20.30~49 범위의 EgressIP가 되어야 함
- 노드의 secondary NIC(bond1)에서 EgressIP가 할당되는지 확인

### 6.4.2 EgressIP 가능 노드 라벨링

OVN-Kubernetes는 노드에 `k8s.ovn.org/egress-assignable=""` 라벨이 있어야 EgressIP를 할당합니다.

```bash
# podpool1 노드에 라벨 부여
oc label node podpool1-worker-0 k8s.ovn.org/egress-assignable=""
oc label node podpool1-worker-1 k8s.ovn.org/egress-assignable=""

# 확인
oc get nodes -l k8s.ovn.org/egress-assignable
```

### 6.4.3 EgressIP CR 작성

```bash
cat > 34-egressip-jboss.yaml <<'EOF'
apiVersion: k8s.ovn.org/v1
kind: EgressIP
metadata:
  name: egress-jboss
spec:
  egressIPs:
  - 192.168.20.30      # 첫 번째 후보
  - 192.168.20.31      # 두 번째 후보 (HA)

  # 적용 대상 namespace
  namespaceSelector:
    matchLabels:
      egress-network: service

  # 적용 대상 Pod (선택)
  # podSelector:
  #   matchLabels:
  #     egress: yes
EOF

oc apply -f 34-egressip-jboss.yaml
```

### 6.4.4 namespace 라벨 부여

```bash
oc label namespace jboss-test egress-network=service
```

### 6.4.5 EgressIP 동작 확인

```bash
# EgressIP가 어느 노드에 할당되었는지
oc get egressip
```

기대:
```
NAME             EGRESSIPS         ASSIGNED NODE         ASSIGNED EGRESSIPS
egress-jboss     192.168.20.30,    podpool1-worker-0,    192.168.20.30,
                 192.168.20.31     podpool1-worker-1     192.168.20.31
```

각 EgressIP가 podpool1 노드에 분산 할당되었음을 확인합니다.

```bash
# 노드에서 실제 IP 부여 확인
oc debug node/podpool1-worker-0 -- chroot /host ip addr | grep 192.168.20
```

기대:
- `192.168.20.61` (호스트 IP, 변경 없음)
- `192.168.20.30` (EgressIP, OVN이 동적 추가)

### 6.4.6 Pod에서 outbound 검증

```bash
# jboss-test namespace에서 외부로 나가는 Pod
oc -n jboss-test run egress-test --image=mirror.ocp1.example.com:8443/ubi9/ubi:latest \
  -- sleep 3600

oc -n jboss-test exec -it egress-test -- bash

# Pod 내부에서:
# 외부 IP를 echo해주는 서비스에 접근 (Bastion에 simple echo server 가정)
curl http://192.168.20.10:8081/whoami
# 기대 응답: "your IP: 192.168.20.30" (또는 .31)
```

**검증 포인트:**
- Pod의 source IP가 192.168.20.30 또는 192.168.20.31 (EgressIP)
- 192.168.20.61 (노드 IP)이 아님
- 192.168.10.x (Infra) 가 아님

### 6.4.7 EgressIP가 동작하지 않을 때

OpenShift 버전과 환경에 따라 EgressIP가 secondary NIC에서 동작하지 않을 수 있습니다. 그 경우 fallback 옵션을 검토합니다.

| 옵션 | 설명 |
|---|---|
| **EgressIP 기본** | OVN-Kubernetes가 자동 관리, 권장 |
| **EgressService** | Service 단위 egress (LB 통과 시) |
| **OutboundOptions** (network operator) | 클러스터 전역 egress 설정 |
| **정책 라우팅** | MachineConfig로 노드 OS에 `ip rule` 적용 |
| **외부 NAT** | Service망 방화벽에서 NAT 처리 |

> **PoC 결과를 문서에 기록**
> 본 PoC에서 EgressIP의 secondary NIC 할당이 안정적으로 동작했는지 결과를 문서에 남겨야, 운영 환경 결정 시 참고할 수 있습니다.
> - 동작함: 권장안으로 채택
> - 동작 불안정: 정책 라우팅 또는 외부 NAT으로 우회

---

## 6.5 종합 검증 체크리스트

본 PoC 전체가 의도대로 동작하는지 일괄 점검하는 체크리스트입니다.

### 6.5.1 인프라 헬스

```bash
# 1. 모든 노드 Ready
oc get nodes

# 2. 모든 MCP UPDATED=True
oc get mcp

# 3. 모든 ClusterOperator Available, Not Degraded
oc get co

# 4. 모든 시스템 Pod Running
oc get pods -A | grep -v 'Running\|Completed' | grep -v 'NAME'

# 5. etcd 상태
oc -n openshift-etcd exec etcd-master-0 -- etcdctl --command-timeout=5s endpoint health --cluster
```

### 6.5.2 네트워크

```bash
# 1. machineNetwork 적용 확인 (모든 노드 InternalIP가 Infra)
oc get nodes -o wide | awk '{print $1, $6}'

# 2. NMState bridge 동작
oc get nncp
oc get nnce

# 3. IngressController 분리
oc get ingresscontroller -n openshift-ingress-operator
oc get pods -n openshift-ingress -o wide

# 4. EgressIP 할당 (검증 시)
oc get egressip

# 5. DNS 해석
oc run dns-test --rm -it --image=mirror.ocp1.example.com:8443/ubi9/ubi:latest -- \
  nslookup kubernetes.default
```

### 6.5.3 Virtualization

```bash
# 1. HyperConverged Available
oc get hyperconverged -n openshift-cnv

# 2. virt-handler가 ap/db에만 (4개)
oc get pods -n openshift-cnv -l kubevirt.io=virt-handler -o wide

# 3. VM 동작 (생성한 경우)
oc get vm -A
oc get vmi -A
```

### 6.5.4 Storage

```bash
# 1. PV 가용성
oc get pv

# 2. PVC 동작 (테스트 PVC 생성 후 확인)
oc get pvc -A

# 3. CDI Pod 동작
oc get pods -n openshift-cnv | grep cdi
```

### 6.5.5 Migration (MTV)

```bash
# 1. ForkliftController Ready
oc get forkliftcontroller -n openshift-mtv

# 2. Provider 등록
oc get provider -n openshift-mtv

# 3. Plan 검증 가능
oc get plan -n openshift-mtv
```

### 6.5.6 Fencing

```bash
# 1. NHC 동작 중
oc get nhc

# 2. FAR Template 생성됨
oc get fenceagentsremediationtemplate -A

# 3. vBMC 응답 (Bastion에서)
for p in 6230 6231 6232 6233 6234 6235; do
  echo "Port $p:"
  ipmitool -I lanplus -H 127.0.0.1 -p $p -U admin -P 'fenceP@ss!' chassis power status 2>&1 | tail -1
done
```

### 6.5.7 IngressController + DNS 통합

```bash
# 1. 관리 Route (apps 도메인)
curl -k https://console-openshift-console.apps.ocp1.example.com -o /dev/null -w "%{http_code}\n"
# 기대: 200 또는 302

# 2. 업무 Route (svcapps 도메인) - 위에서 만든 hello-service
curl -k https://hello-service-service-test.svcapps.ocp1.example.com -o /dev/null -w "%{http_code}\n"
# 또는 jboss
curl -k https://eap-app-jboss-test.svcapps.ocp1.example.com -o /dev/null -w "%{http_code}\n"
```

---

## 6.6 트러블슈팅 가이드

본 PoC 전체에서 발생할 수 있는 흔한 문제 18가지를 단계별로 정리합니다.

### 6.6.1 설치 단계 문제

**1. Mirror Registry 인증 실패**

증상: `ImagePullBackOff`, 로그에 `unauthorized`

```bash
# 노드에서 mirror registry 인증 정보 확인
oc debug node/master-0 -- chroot /host cat /var/lib/kubelet/config.json | jq '.auths | keys'
# mirror.ocp1.example.com:8443가 있어야 함
```

원인:
- pull-secret에 Mirror Registry 인증 누락
- additionalTrustBundle 누락
- Mirror Registry CA 미신뢰

해결:
```bash
# pull-secret 업데이트
oc set data secret/pull-secret -n openshift-config \
  --from-file=.dockerconfigjson=/path/to/updated-pull-secret.json
```

**2. DNS 해석 실패 (api/api-int/apps)**

증상: 노드가 부팅 후 API에 도달하지 못함

```bash
# 노드에서
nslookup api-int.ocp1.example.com
dig api.ocp1.example.com
```

원인:
- Bastion DNS 미실행
- 정방향/역방향 zone 파일 오류
- 노드 NetworkManager의 DNS 설정 누락

해결:
- Bastion: `systemctl status named`, `named-checkzone`
- 노드: `nmcli con show <connection>`에서 `ipv4.dns` 확인

**3. LB 6443/22623 구성 오류**

증상: bootstrap-complete 대기 무한 지연

```bash
# Bastion에서 LB backend 상태
curl -s -u admin:RedHat123! 'http://192.168.10.10:9000/;csv' | grep -E '6443|22623'
```

원인:
- HAProxy frontend bind IP가 노드의 실제 IP와 다름
- backend의 master IP 오타
- SELinux의 `haproxy_connect_any` 미설정

해결:
- `haproxy -c -f /etc/haproxy/haproxy.cfg` 검증
- `setsebool -P haproxy_connect_any 1`

**4. bootstrap-complete 30분 이상 지연**

증상: `openshift-install wait-for bootstrap-complete` 진행 없음

```bash
# bootstrap에 SSH
ssh core@192.168.10.30
sudo journalctl -u bootkube -f
sudo journalctl -u kubelet -f
```

원인:
- master에서 bootstrap에 ignition 못 받음
- release image pull 실패
- container engine 시작 실패

**5. worker CSR 자동 승인 누락**

증상: worker가 NotReady, `oc get csr`에 Pending 대량

```bash
oc get csr | grep Pending
oc get csr -o name | xargs oc adm certificate approve
```

근본 원인:
- worker InternalIP가 machineNetwork 밖
- 노드 hostname이 DNS에서 역방향 해석 안 됨
- 시간 동기화 어긋남

**6. copy-network 후 bond 설정 미반영**

증상: 재부팅 후 노드 IP가 사라짐, DHCP 시도

원인: `--copy-network` 옵션 누락

해결:
- Live ISO 다시 부팅 → 네트워크 구성 → `--copy-network` 옵션 포함하여 재설치

### 6.6.2 Day-2 단계 문제

**7. MCP 업데이트 실패**

증상: MCP `UPDATING=True` 멈춤, 노드가 SchedulingDisabled 상태로 정지

```bash
oc get mcp
oc describe mcp <pool>
oc get pod -n openshift-machine-config-operator
```

원인:
- 잘못된 MachineConfig (잘못된 ignition 형식, 디스크 wipe 등)
- MCD(Machine Config Daemon) 에러

해결:
- 문제 MC를 식별하여 삭제 또는 수정
- 노드가 부팅 단계에서 멈추면 콘솔로 직접 접근하여 oc unpause

**8. MachineConfig 적용 후 노드 NotReady**

증상: 재부팅 후 노드가 NotReady, kubelet 시작 안 됨

```bash
# 노드 SSH
ssh core@<node>
sudo journalctl -u kubelet -b
sudo journalctl -u crio -b
```

원인:
- 잘못된 systemd unit
- 잘못된 kernel arg (예: `root=` 오류)
- 파일 권한 충돌

해결:
- 노드 부팅 시 GRUB 메뉴에서 이전 OS 버전 선택 (RHCOS는 두 버전 보관)
- MC 수정 후 다시 적용

### 6.6.3 네트워크 단계 문제

**9. NNCP 적용 후 네트워크 단절**

증상: NNCE Failing, 노드 SSH 끊김

원인: 잘못된 NIC 이름 또는 잘못된 bridge 구성

해결: NMState가 1분 후 자동 rollback. 수동 개입 불필요.

```bash
oc get nnce
# Failing 상태였다가 Rolled back으로 변경됨
```

**10. NAD 연결 후 VM 네트워크 미동작**

증상: VM이 Service IP에 ping 안 됨

```bash
# VM 콘솔
ip addr show eth1
ping 192.168.20.1
```

원인:
- bridge가 안 만들어짐
- 외부 스위치의 VLAN/trunk 설정 누락
- bridge가 STP forwarding 상태 아님

해결:
- `bridge link` 확인
- 스위치 측 trunk 설정
- `bridge link set dev <port> learning off` (PoC 단순화)

**11. 업무 IngressController가 podpool1에 배치되지 않음**

증상: `router-service` Pod가 다른 노드에 배치

```bash
oc get pods -n openshift-ingress -o wide -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=service
```

원인: IngressController CR의 `nodePlacement` 누락

해결: Part 5.9 참조하여 nodePlacement 추가

### 6.6.4 스토리지 단계 문제

**12. NAS StorageClass PVC Pending**

증상: PVC가 Pending 상태로 멈춤

```bash
oc describe pvc <name>
```

원인:
- 정적 PV 부족 (수동 PV 방식)
- accessMode 매칭 안 됨 (PV는 RWX, PVC는 RWO 등)
- StorageClass 이름 불일치

해결:
- PV 추가 생성
- accessMode 일치 확인

**13. CDI DataVolume import 실패**

증상: DataVolume이 ImportInProgress에서 Failed로 변경

```bash
oc get dv -A
oc describe dv <name>
oc logs -n <namespace> -l app=containerized-data-importer
```

원인:
- import source URL 접근 불가
- PVC 권한 문제
- 디스크 크기 부족

### 6.6.5 시나리오 단계 문제

**14. MTV Migration Plan 실패**

(6.1.10 참조)

**15. FAR fencing 실패**

(6.2.9 참조)

**16. vBMC 연결 실패**

증상: FAR Pod 로그에 `Connection refused`

```bash
# Bastion에서
vbmc list
# running 상태인지 확인

systemctl status vbmcd

# 수동 IPMI 테스트
ipmitool -I lanplus -H 127.0.0.1 -p 6230 -U admin -P 'fenceP@ss!' chassis power status
```

원인:
- vBMC 데몬 미실행
- 노드별 vBMC 인스턴스 미시작
- 방화벽

**17. additionalTrustBundle 누락**

증상: 설치는 성공했지만 일부 컴포넌트가 mirror에 접근 못 함

```bash
# CA bundle 확인
oc get configmap -n openshift-config user-ca-bundle -o yaml
```

해결:
```bash
# CA 인증서 추가
oc create configmap user-ca-bundle \
  --from-file=ca-bundle.crt=/path/to/mirror-ca.crt \
  -n openshift-config \
  --dry-run=client -o yaml | oc apply -f -

# Proxy 리소스 패치
oc patch proxy/cluster --type=merge -p '{"spec":{"trustedCA":{"name":"user-ca-bundle"}}}'
```

**18. pull-secret 병합 오류**

증상: pull-secret 업데이트 후 일부 노드가 이미지 pull 실패

```bash
# 클러스터 pull-secret 확인
oc get secret pull-secret -n openshift-config -o json | \
  jq -r '.data.".dockerconfigjson"' | base64 -d | jq
```

해결: 모든 auth 항목이 포함되었는지 확인 후 secret 재생성

---

## 6.7 PoC → 프로덕션 전환 체크리스트

본 PoC 완료 후 프로덕션 환경으로 전환할 때 변경해야 할 항목을 일괄 정리합니다.

### 6.7.1 노드/MCP

| 항목 | PoC | 프로덕션 | 작업 |
|---|---|---|---|
| infra MCP | 없음 | 3대 신설 | 노드 추가, MCP 생성, 라벨링 |
| Default IngressController 배치 | ap MCP | infra MCP | IngressController CR nodePlacement 변경 |
| Monitoring 스택 | 기본 위치 | infra MCP | cluster-monitoring-config ConfigMap 수정 |
| Logging 스택 | (선택) | infra MCP | ClusterLogging CR 수정 |
| Image Registry | 기본 | infra MCP | Config CR nodePlacement 수정 |
| worker 라벨 | 유지 | infra/ap/db/podpool1에서 제거 | `oc label node <n> node-role.kubernetes.io/worker-` |

### 6.7.2 Load Balancer

| 항목 | PoC | 프로덕션 | 작업 |
|---|---|---|---|
| Default Ingress LB | Bastion HAProxy | 관리망 LB 또는 Bastion 이중화 | 별도 LB 구축 |
| Service Ingress LB | Bastion HAProxy (별도 frontend) | 별도 물리 L4/L7 | 물리 장비 도입, VIP 이전 |
| LB HA | 단일 Bastion | LB 장비 이중화 | VRRP 또는 장비별 HA |
| API LB | Bastion HAProxy | 관리망 LB | 별도 LB로 이전 |

### 6.7.3 Bastion

| 항목 | PoC | 프로덕션 | 작업 |
|---|---|---|---|
| Bastion NIC | Infra + Service | Infra만 | Service NIC 제거, Service VIP 이전 |
| Mirror Registry | Bastion 통합 | 별도 노드 또는 HA Quay | Mirror Registry 분리 |
| DNS | Bastion BIND | 기업 DNS 인프라 | DNS 이전 또는 forward 설정 |

### 6.7.4 스토리지

| 항목 | PoC | 프로덕션 | 작업 |
|---|---|---|---|
| NAS 망 | Infra와 통합 | 별도 망 분리 검토 | NAS 전용 NIC 추가 |
| NFS Provisioning | 수동 PV | Dynamic CSI Driver | CSI Driver 도입 |
| Monitoring Storage | emptyDir 또는 NFS | block storage | ODF, Trident 등 도입 |
| etcd backup | 수동 또는 cron | 자동화 + 분리 보관 | EtcdBackup CR + offsite |

### 6.7.5 보안 및 인증서

| 항목 | PoC | 프로덕션 | 작업 |
|---|---|---|---|
| API 인증서 | 자체 서명 | 사내 CA | API 인증서 교체 |
| Ingress 인증서 (default) | 자체 서명 | 사내 CA | apps 와일드카드 인증서 교체 |
| Ingress 인증서 (service) | 자체 서명 | 사내 CA | svcapps 와일드카드 인증서 교체 |
| kubeadmin | 활성 | 비활성화 | OAuth + LDAP/SSO 도입 후 kubeadmin 삭제 |
| Pod 권한 | 기본 SCC | 엄격한 SCC | 워크로드별 SCC 정의 |

### 6.7.6 Virtualization

| 항목 | PoC | 프로덕션 | 작업 |
|---|---|---|---|
| HyperConverged infra.nodePlacement | master | infra MCP | nodeSelector 변경 |
| LiveMigration 네트워크 | Infra 공유 | 별도 migration network | NAD 추가, HyperConverged 패치 |
| CPU pinning | 미적용 | DB VM에 적용 | KubeletConfig + HyperConverged 수정 |
| Hugepages | 미적용 | 필요 VM에 적용 | KubeletConfig + VM spec |

### 6.7.7 네트워크

| 항목 | PoC | 프로덕션 | 작업 |
|---|---|---|---|
| EgressIP | PoC 검증 | 운영 적용 | namespace별 EgressIP 매핑 |
| 외부 방화벽 | 단순 허용 | 세분화된 정책 | 망별 ACL 작성 |
| Service Mesh mTLS | 선택 | 권장 활성화 | PeerAuthentication |

### 6.7.8 모니터링/로깅

| 항목 | PoC | 프로덕션 | 작업 |
|---|---|---|---|
| User Workload Monitoring | 미활성화 | 활성화 | cluster-monitoring-config 패치 |
| Cluster Logging | 선택 | 권장 활성화 | Loki Operator 설치 |
| Alertmanager 통합 | 기본 | 사내 알림 (이메일, Slack 등) | AlertmanagerConfig |

### 6.7.9 운영 절차

| 항목 | PoC | 프로덕션 | 작업 |
|---|---|---|---|
| etcd 백업 | 수동 | 자동 + 검증 | CronJob 또는 EtcdBackup CR |
| 업그레이드 절차 | 미수립 | 정기 z-stream 패치 | Update Service 또는 수동 절차 |
| Disaster Recovery | 미수립 | etcd 복구, 재설치 절차 | DR 매뉴얼 작성 |
| RBAC | 기본 (kubeadmin) | 사용자별 권한 | Group/Role 설계 |

---

## 6.8 학습 자료 및 다음 단계

### 6.8.1 공식 문서

학습을 더 깊이 진행하려면 다음 공식 문서를 참고하시기 바랍니다.

| 주제 | 문서 |
|---|---|
| OpenShift 4.20 설치 | docs.redhat.com/ko/documentation/openshift_container_platform/4.20/html/installing |
| OpenShift Virtualization | docs.redhat.com (Virtualization 섹션) |
| Migration Toolkit for Virtualization | docs.redhat.com (Migration Toolkit) |
| OVN-Kubernetes | OpenShift Networking 문서 |
| oc-mirror v2 | docs.redhat.com (Disconnected installation mirroring) |

### 6.8.2 학습 순서 권장

본 가이드를 마친 후 다음 순서로 심화 학습을 권장합니다.

1. **Helm/Operator 개발 학습** - Operator SDK로 자체 Operator 작성
2. **Pipeline as Code** - Tekton + GitOps 통합 워크플로우
3. **Service Mesh 심화** - mTLS, 트래픽 분할, 카나리 배포
4. **Observability** - Prometheus + Grafana 커스텀 대시보드, Loki 로그 쿼리
5. **Disaster Recovery** - etcd 복구, 클러스터 백업/복원 시나리오 실습
6. **Multi-cluster** - ACM (Advanced Cluster Management)으로 다중 클러스터 관리

### 6.8.3 본 PoC 산출물 활용

본 PoC를 완료하면 다음 산출물이 남습니다.

1. **설치 자동화 스크립트** (`~/build-install-config.sh` 등)
2. **모든 매니페스트 파일** (`~/openshift-day2/`)
3. **시나리오 검증 결과** (EgressIP 동작 여부, MTV 마이그레이션 결과, FAR 동작 시간 등)
4. **트러블슈팅 기록**

이 산출물들은 운영 환경 구축의 출발점이 됩니다. PoC 단계에서 만난 모든 문제와 해결책을 기록해두면, 운영 도입 시점에 시행착오를 크게 줄일 수 있습니다.

---

## 6.9 Part 6 학습 점검

다음 질문에 답할 수 있다면 Part 6를 충분히 학습한 것입니다.

1. MTV가 사용하는 5가지 핵심 CR (Provider, NetworkMap, StorageMap, Plan, Migration)의 역할과 관계는?
2. MTV의 warm migration과 cold migration의 차이는? 각각 언제 적합한가?
3. fencing이 필요한 워크로드 종류 세 가지는?
4. NHC와 FAR의 역할 분담은? 둘이 어떻게 협력하는가?
5. vBMC가 PoC 환경에서 필요한 이유는? 프로덕션에서는 어떻게 대체되는가?
6. NHC의 `minHealthy: 51%` 설정의 의미는? 너무 낮게 두면 어떤 위험이 있는가?
7. namespace에 `openshift.io/node-selector` 어노테이션을 사용하는 것의 장점은?
8. EgressIP가 동작하려면 노드에 어떤 라벨이 필요한가? 그리고 OVN-Kubernetes가 EgressIP를 어디에 할당하는가?
9. EgressIP가 secondary NIC에서 동작하지 않을 때의 대안 세 가지는?
10. MCP 업데이트 실패 시 흔한 세 가지 원인은?
11. PoC → 프로덕션 전환 시 가장 큰 변화 세 가지는?
12. 본 가이드의 설계 원칙(관리/서비스 트래픽 분리)이 실제 운영에서 가져오는 이점 세 가지는?

---

## 6.10 가이드 종료

본 가이드는 다음 6개 파트로 구성되었습니다.

| Part | 주제 | 핵심 내용 |
|---|---|---|
| 1 | 아키텍처 및 네트워크 설계 | 관리/서비스 트래픽 분리, machineNetwork, IngressController 설계 |
| 2 | Bastion 및 Load Balancer | DNS, HAProxy, Mirror Registry, NTP |
| 3 | 폐쇄망 미러링 및 설치 | oc-mirror v2 2단계 흐름, install-config, RHCOS 설치 |
| 4 | Day-2 MCP 및 MachineConfig | MCP 분리, chrony/kdump/multipath, 인증서, etcd 백업 |
| 5 | Operator 및 Virtualization | NMState, OpenShift Virtualization, NAD, IngressController 분리 |
| 6 | 시나리오 테스트 및 운영 | MTV, FAR/vBMC, 워크로드 배치, 트러블슈팅 |

각 파트는 단순한 절차 모음이 아니라 **"왜 그렇게 하는가"를 학습할 수 있는 교육자료**로 작성되었습니다. 각 파트 끝의 학습 점검 질문에 모두 답할 수 있다면, 본 PoC 환경의 모든 결정 배경을 이해한 것입니다.

PoC를 실제로 수행하시면서 발견한 추가 사항은 본 문서에 보완 기록으로 남겨, 다음 운영 환경 구축의 디딤돌로 활용하시기 바랍니다.

---

*가이드 끝.*
