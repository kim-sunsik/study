# Part 4. Day-2 MCP 분리 및 MachineConfig

> **이 파트의 목적**
> Part 3에서 설치 완료한 클러스터는 master/worker 두 개의 기본 MCP만 가진 상태입니다. 본 파트에서는 워크로드 격리 원칙(VM 전용 vs 컨테이너 전용)을 실제 노드 그룹으로 구현하기 위해 ap, db, podpool1 세 개의 MCP로 worker를 분리하고, 각 MCP에 차등 정책(chrony, kdump, multipath)을 MachineConfig로 적용합니다.
>
> 또한 운영 환경 진입 전에 필수적인 인증서 관리, etcd 백업 정책, 노드 디스크 설계를 다룹니다.
>
> 이 파트를 학습한 후 학습자는 MCP/MachineConfig의 동작 원리, Day-2 운영의 핵심 절차, 그리고 변경 작업이 클러스터에 미치는 영향(rolling reboot)을 설명할 수 있어야 합니다.

---

## 4.1 MachineConfigPool과 MachineConfig 개념

### 4.1.1 MachineConfigPool (MCP)이란

**MCP는 동일한 OS 설정을 공유하는 노드들의 논리적 그룹**입니다. OpenShift 노드의 OS(RHCOS) 설정은 Kubernetes 컨테이너처럼 선언적으로 관리되는데, 그 단위가 MCP입니다.

기본 MCP:
- `master`: control plane 노드들
- `worker`: 모든 worker 노드들

본 PoC에서 분리할 MCP:
- `master`: 변경 없음
- `ap`: AP성 VM 노드 (ap-worker-0, ap-worker-1)
- `db`: DB성 VM 노드 (db-worker-0, db-worker-1)
- `podpool1`: 컨테이너 워크로드 노드 (podpool1-worker-0, podpool1-worker-1)

### 4.1.2 MachineConfig (MC)란

**MC는 한 가지 OS 설정 변경을 정의하는 선언적 리소스**입니다. systemd unit, 파일, kernel argument, hostname 등을 모두 MC로 표현할 수 있습니다.

MC의 핵심 특징:
- **선언적**: 원하는 상태를 정의하면 MCO(Machine Config Operator)가 모든 노드에 적용
- **노드 재부팅 유발**: 대부분의 MC 변경은 노드 재부팅을 동반 (kernel arg, systemd 등)
- **MCP 단위 적용**: MC는 label selector로 적용 대상 MCP를 지정
- **누적적**: 여러 MC가 같은 MCP에 적용되면 모두 병합되어 `rendered-<mcp>-<hash>` 형태로 노드에 배포

### 4.1.3 MCO (Machine Config Operator)

MCO는 MC를 노드에 적용하는 OpenShift Operator입니다. 동작 흐름:

```
1. 사용자가 MC 생성
   ↓
2. MCO가 해당 MCP에 속한 MC들을 병합 → rendered-<mcp>-<hash> 생성
   ↓
3. MCP 상태가 UPDATING → True
   ↓
4. MCO가 노드 한 대씩 처리 (maxUnavailable=1 기본)
   - 노드 cordon (스케줄링 중단)
   - 워크로드 drain
   - 새 설정 적용
   - 재부팅
   - 노드 Ready 대기
   - uncordon
   ↓
5. 모든 노드 완료 시 MCP UPDATED=True
```

> **MC 변경의 비용을 항상 인식하라**
> MC를 하나 추가하거나 수정할 때마다 해당 MCP의 모든 노드가 한 대씩 재부팅됩니다.
> - master MCP: 3대 재부팅 → 약 20~30분 + etcd quorum 주의
> - worker MCP: 노드 수만큼 재부팅 → 노드당 5~10분
> - PoC라도 운영 중 워크로드가 있다면 drain 영향 검토 필수
>
> **여러 MC를 만들 계획이라면 가능한 한 모아서 한 번에 적용**하여 rolling reboot 횟수를 줄이는 것이 좋습니다.

### 4.1.4 본 PoC의 MCP 분리 근거

ap/db/podpool1을 별도 MCP로 분리하는 이유를 명확히 합니다.

**ap와 db를 분리하는 이유**

| 근거 | 설명 |
|---|---|
| 장애 영향 범위 분리 | AP성 VM 장애가 DB성 VM에 영향 주지 않음 |
| 차등 정책 적용 | CPU pinning, hugepages, reserved CPU를 다르게 설정 가능 |
| kdump 정책 분리 | DB는 dump 보관, AP는 즉시 재부팅 같은 정책 |
| 패치 윈도우 분리 | AP와 DB의 정기 점검 시간 분리 운영 |
| LiveMigration 정책 | MCP 내부만 허용/MCP 경계 넘기 허용 등 명시적 정의 |

**podpool1을 별도로 두는 이유**

| 근거 | 설명 |
|---|---|
| 워크로드 성격 차이 | VM과 컨테이너는 자원 패턴, 디스크 I/O, 네트워크 사용이 다름 |
| Service NIC 사용 | podpool1만 Service NIC에 IP를 가짐 (Part 1.6.3) |
| Service IngressController 배치 | 업무용 IngressController가 podpool1에만 배치됨 |
| multipath 등 스토리지 정책 | 컨테이너 노드는 일반적으로 multipath 불필요 |

> **PoC vs 프로덕션**
> 프로덕션에서는 추가로 `infra` MCP를 신설하여 Default IngressController, monitoring, logging, image registry를 격리합니다. infra 노드는 OpenShift 구독에서 면제됩니다. (Part 1.10 참조)
> 본 PoC는 자원 절약을 위해 infra MCP 없이 Default IngressController를 ap MCP에 임시 배치합니다.

---

## 4.2 MCP 분리 절차

### 4.2.1 분리 전 상태 확인

```bash
export KUBECONFIG=~/openshift-install-work/auth/kubeconfig

# 현재 MCP 확인
oc get mcp
```

기대 (설치 직후 상태):
```
NAME     CONFIG                       UPDATED   UPDATING   DEGRADED
master   rendered-master-XXXXXXXX     True      False      False
worker   rendered-worker-XXXXXXXX     True      False      False
```

```bash
# 현재 노드별 라벨 확인
oc get nodes --show-labels
```

worker 노드에는 다음 라벨이 있을 것입니다:
- `node-role.kubernetes.io/worker=`
- `kubernetes.io/hostname=<hostname>`
- 기타 자동 라벨

### 4.2.2 노드 라벨링

각 노드에 역할 라벨을 추가합니다.

```bash
# AP 노드
oc label node ap-worker-0 node-role.kubernetes.io/ap=""
oc label node ap-worker-1 node-role.kubernetes.io/ap=""

# DB 노드
oc label node db-worker-0 node-role.kubernetes.io/db=""
oc label node db-worker-1 node-role.kubernetes.io/db=""

# PodPool1 노드
oc label node podpool1-worker-0 node-role.kubernetes.io/podpool1=""
oc label node podpool1-worker-1 node-role.kubernetes.io/podpool1=""
```

> **라벨 이름의 의미**
> `node-role.kubernetes.io/<role>` 형식의 라벨은 OpenShift가 인식하는 노드 역할 라벨입니다. 이 라벨이 있으면 `oc get nodes`의 ROLES 컬럼에 `<role>`이 추가로 표시됩니다. 또한 nodeSelector에서 이 라벨로 Pod를 특정 노드에만 스케줄링할 수 있습니다.

라벨 확인:

```bash
oc get nodes
```

기대:
```
NAME                STATUS   ROLES                  AGE   VERSION
master-0            Ready    control-plane,master   2h    v1.31.x
master-1            Ready    control-plane,master   2h    v1.31.x
master-2            Ready    control-plane,master   2h    v1.31.x
ap-worker-0         Ready    ap,worker              1h    v1.31.x
ap-worker-1         Ready    ap,worker              1h    v1.31.x
db-worker-0         Ready    db,worker              1h    v1.31.x
db-worker-1         Ready    db,worker              1h    v1.31.x
podpool1-worker-0   Ready    podpool1,worker        1h    v1.31.x
podpool1-worker-1   Ready    podpool1,worker        1h    v1.31.x
```

각 노드의 ROLES에 `ap,worker`, `db,worker`, `podpool1,worker` 형태로 표시되면 라벨이 정상 적용된 것입니다.

### 4.2.3 신규 MCP 생성

ap, db, podpool1 MCP를 생성합니다.

```bash
mkdir -p ~/openshift-day2/mcp
cd ~/openshift-day2/mcp
```

**ap MCP 정의:**

```bash
cat > 01-mcp-ap.yaml <<'EOF'
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: ap
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - ap
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/ap: ""
EOF
```

**db MCP 정의:**

```bash
cat > 02-mcp-db.yaml <<'EOF'
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: db
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - db
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/db: ""
EOF
```

**podpool1 MCP 정의:**

```bash
cat > 03-mcp-podpool1.yaml <<'EOF'
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: podpool1
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - podpool1
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/podpool1: ""
EOF
```

### 4.2.4 machineConfigSelector 해설

```yaml
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - ap
```

이 설정은 **"이 MCP는 `worker` 역할 MC와 `ap` 역할 MC를 모두 포함한다"**는 의미입니다.

| 의도 | 결과 |
|---|---|
| `worker` 포함 | 기존 worker MC(kubelet, container runtime 등 클러스터 기본 설정)를 그대로 상속 |
| `ap` 추가 | 추가로 `ap` 전용 MC를 적용 가능 |

만약 `worker`를 빼고 `ap`만 두면, 클러스터 기본 worker 설정이 적용되지 않아 노드가 정상 동작하지 않습니다.

> **흔한 실수: machineConfigSelector에서 worker 누락**
> ```yaml
> # 잘못된 예
> machineConfigSelector:
>   matchExpressions:
>   - key: machineconfiguration.openshift.io/role
>     operator: In
>     values:
>     - ap     # worker가 없음!
> ```
> 이렇게 하면 기본 kubelet 설정, CNI, 인증서 등이 모두 빠진 빈 MC가 노드에 적용되어 노드가 NotReady됩니다.

### 4.2.5 nodeSelector 해설

```yaml
spec:
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/ap: ""
```

이 설정은 **"이 MCP는 `node-role.kubernetes.io/ap` 라벨이 있는 노드를 관리한다"**는 의미입니다.

라벨 매칭 동작:
- 노드에 `node-role.kubernetes.io/ap=""` 라벨이 있으면 ap MCP에 자동 편입
- 노드에 `node-role.kubernetes.io/worker=""` 라벨만 있으면 worker MCP가 관리
- 두 라벨이 모두 있으면 **더 구체적인 MCP가 우선** (ap MCP가 우선, worker MCP는 무시)

> **노드는 한 번에 하나의 MCP에만 속한다**
> 라벨을 여러 개 가져도 MCP는 노드별로 단 하나만 책임집니다. nodeSelector가 매칭되는 MCP 중 우선순위가 결정되며, 기본 worker MCP는 다른 모든 사용자 정의 MCP보다 낮은 우선순위를 갖습니다.

### 4.2.6 MCP 적용

```bash
oc apply -f 01-mcp-ap.yaml
oc apply -f 02-mcp-db.yaml
oc apply -f 03-mcp-podpool1.yaml
```

기대 출력:
```
machineconfigpool.machineconfiguration.openshift.io/ap created
machineconfigpool.machineconfiguration.openshift.io/db created
machineconfigpool.machineconfiguration.openshift.io/podpool1 created
```

### 4.2.7 MCP 편입 확인

MCP가 생성되면 nodeSelector가 매칭되는 노드들이 자동으로 새 MCP로 편입됩니다.

```bash
oc get mcp
```

기대:
```
NAME       CONFIG                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT
ap         rendered-ap-XXXXXXXX           True      False      False      2              2
db         rendered-db-XXXXXXXX           True      False      False      2              2
master     rendered-master-XXXXXXXX       True      False      False      3              3
podpool1   rendered-podpool1-XXXXXXXX     True      False      False      2              2
worker     rendered-worker-XXXXXXXX       True      False      False      0              0
```

**핵심 검증 포인트:**
- ap MCP: 2대
- db MCP: 2대
- podpool1 MCP: 2대
- worker MCP: 0대 (모든 노드가 전용 MCP로 이동됨)
- 모든 MCP가 UPDATED=True

> **MCP 생성 직후 rolling reboot 여부**
> MCP만 새로 만들고 추가 MC가 없다면 **rolling reboot은 발생하지 않습니다**. 노드의 rendered config는 같은 worker 기본 설정에서 변하지 않기 때문입니다.
> 실제 rolling reboot은 ap/db/podpool1 MCP에 전용 MC(chrony, kdump, multipath 등)를 추가했을 때 발생합니다.

### 4.2.8 worker 라벨 정책 결정

각 노드는 현재 `worker`와 `<역할>` 두 라벨을 모두 가집니다. worker 라벨을 유지할지 제거할지 결정해야 합니다.

**옵션 A: worker 라벨 유지 (PoC 권장)**

```bash
# 변경 없음 - 현재 상태 유지
oc get nodes -L node-role.kubernetes.io/worker
```

장점:
- 기본 worker MCP 자동 호환
- 라벨로 워크로드를 worker 전체에 배포 가능 (예: DaemonSet)
- 기존 chart, operator의 nodeSelector 호환성 유지

단점:
- "이 노드는 ap이면서 worker"라는 모호한 이중 정체성
- 향후 infra MCP 도입 시 라벨 제거 필요

**옵션 B: worker 라벨 제거 (프로덕션 권장)**

```bash
# 모든 전용 MCP 노드에서 worker 라벨 제거
for node in ap-worker-0 ap-worker-1 db-worker-0 db-worker-1 \
            podpool1-worker-0 podpool1-worker-1; do
  oc label node $node node-role.kubernetes.io/worker-
done

oc get nodes
# ROLES 컬럼: ap, db, podpool1 (worker 없음)
```

장점:
- 명확한 단일 역할
- OpenShift 구독 모델 (특히 infra 노드 적용 시 필수)
- 의도되지 않은 워크로드 배치 방지

단점:
- DaemonSet 등 기본 worker 대상 워크로드의 nodeSelector 조정 필요
- 일부 Operator가 worker 라벨을 가정할 수 있음

> **본 PoC 권장: 옵션 A (worker 라벨 유지)**
> PoC 환경에서는 OpenShift Virtualization, MTV, Tekton 등 여러 Operator를 설치하고 검증하는데, 일부 Operator는 기본적으로 `worker` 노드를 대상으로 합니다. worker 라벨을 유지하면 호환성 문제를 줄일 수 있습니다.
> 프로덕션 전환 시점에 infra MCP 도입과 함께 worker 라벨 제거를 일괄 수행하는 것을 권장합니다.

---

## 4.3 MachineConfig 적용 - chrony

### 4.3.1 chrony MC의 목적

Part 2.6에서 Bastion을 NTP 서버로 구성했습니다. 각 OpenShift 노드는 Bastion(192.168.10.10)을 시간 동기화 소스로 사용해야 합니다.

RHCOS 기본 chrony 설정은 pool.ntp.org 등 공용 NTP를 사용하므로 폐쇄망에서는 동작하지 않습니다. **MachineConfig로 chrony 설정을 Bastion 기반으로 변경**해야 합니다.

### 4.3.2 chrony 설정 파일 작성

각 MCP별로 거의 동일한 내용이지만 명시적으로 분리하여 작성합니다.

```bash
mkdir -p ~/openshift-day2/mc-chrony
cd ~/openshift-day2/mc-chrony
```

**chrony.conf 내용 (모든 노드 공통):**

```bash
cat > chrony.conf <<'EOF'
# Bastion NTP 서버 지정
server 192.168.10.10 iburst

# 시스템 시간이 1초 이상 차이날 경우 즉시 보정 (시작 시 최대 3회)
makestep 1.0 3

# 시스템 RTC(하드웨어 시계)와 동기화
rtcsync

# 시간 드리프트 보정 파일
driftfile /var/lib/chrony/drift

# 로그 디렉토리
logdir /var/log/chrony

# 외부에 시간 제공 안 함
# allow 192.168.0.0/16   # 클라이언트가 아니므로 주석
EOF
```

**base64 인코딩:**

```bash
# chrony.conf를 base64 인코딩
CHRONY_B64=$(base64 -w0 chrony.conf)
echo $CHRONY_B64
```

### 4.3.3 MC 정의 (master)

```bash
cat > 99-master-chrony.yaml <<EOF
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
      - path: /etc/chrony.conf
        mode: 0644
        overwrite: true
        contents:
          source: data:text/plain;charset=utf-8;base64,${CHRONY_B64}
EOF
```

### 4.3.4 MC 정의 (ap)

```bash
cat > 99-ap-chrony.yaml <<EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: ap
  name: 99-ap-chrony
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
      - path: /etc/chrony.conf
        mode: 0644
        overwrite: true
        contents:
          source: data:text/plain;charset=utf-8;base64,${CHRONY_B64}
EOF
```

### 4.3.5 MC 정의 (db, podpool1)

```bash
# db MC
cat > 99-db-chrony.yaml <<EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: db
  name: 99-db-chrony
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
      - path: /etc/chrony.conf
        mode: 0644
        overwrite: true
        contents:
          source: data:text/plain;charset=utf-8;base64,${CHRONY_B64}
EOF

# podpool1 MC
cat > 99-podpool1-chrony.yaml <<EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: podpool1
  name: 99-podpool1-chrony
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
      - path: /etc/chrony.conf
        mode: 0644
        overwrite: true
        contents:
          source: data:text/plain;charset=utf-8;base64,${CHRONY_B64}
EOF
```

### 4.3.6 MC 적용 및 rolling reboot 모니터링

```bash
# 모든 chrony MC를 한 번에 적용
oc apply -f 99-master-chrony.yaml
oc apply -f 99-ap-chrony.yaml
oc apply -f 99-db-chrony.yaml
oc apply -f 99-podpool1-chrony.yaml

# MCP 상태 모니터링
watch 'oc get mcp; echo "---"; oc get nodes'
```

기대 진행:
1. 각 MCP의 UPDATING 컬럼이 True로 변경
2. 노드가 한 대씩 SchedulingDisabled → NotReady,SchedulingDisabled → Ready 순으로 변경
3. 한 노드가 완료되면 다음 노드 진행
4. 모든 MCP가 UPDATED=True가 되면 완료

**소요 시간 예상 (PoC 9대 기준):**

| MCP | 노드 수 | 소요 시간 |
|---|---|---|
| master | 3 | 약 30~45분 (etcd 안정화 시간 포함) |
| ap | 2 | 약 10~20분 |
| db | 2 | 약 10~20분 |
| podpool1 | 2 | 약 10~20분 |

> **MCP는 병렬 진행됨**
> ap, db, podpool1 MCP는 서로 독립이므로 동시에 rolling reboot이 진행됩니다. master는 별도로 진행됩니다.
> 단, master rolling reboot 중에는 API 서버가 일시적으로 한 대씩 빠지므로 oc 명령이 잠시 끊길 수 있습니다. 이는 정상입니다.

### 4.3.7 적용 확인

```bash
# 한 노드에서 chrony 동기화 상태 확인
oc debug node/master-0 -- chroot /host chronyc tracking

# 기대:
# Reference ID    : C0A80A0A (192.168.10.10)
# Stratum         : 3
# System time     : 0.000XXX seconds slow/fast of NTP time
```

```bash
# 모든 노드 일괄 확인
for node in $(oc get nodes -o name); do
  echo "=== $node ==="
  oc debug $node -- chroot /host chronyc sources -v 2>/dev/null | grep -E "192\.168\.10\.10|Reference ID"
done
```

---

## 4.4 MachineConfig 적용 - kdump

### 4.4.1 kdump의 목적

kdump는 커널 패닉 발생 시 메모리 덤프를 디스크에 저장하는 메커니즘입니다. 운영 장애 분석을 위해 권장됩니다.

| 항목 | 필요성 |
|---|---|
| master | etcd 장애 분석에 필수 |
| ap/db (VM 노드) | VM 호스트 커널 장애 분석 |
| podpool1 | 컨테이너 노드 커널 장애 분석 |

### 4.4.2 kdump 설정 방법

kdump 활성화에는 다음이 필요합니다.

1. **kernel arg 추가**: `crashkernel=<size>` (재부팅 시 메모리 예약)
2. **kdump.service 활성화**: systemd로 부팅 시 자동 시작
3. **덤프 저장 경로 설정**: 기본 `/var/crash`

### 4.4.3 MC 정의 (master)

```bash
mkdir -p ~/openshift-day2/mc-kdump
cd ~/openshift-day2/mc-kdump
```

```bash
cat > 99-master-kdump.yaml <<'EOF'
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-kdump
spec:
  kernelArguments:
  - crashkernel=512M
  config:
    ignition:
      version: 3.4.0
    systemd:
      units:
      - name: kdump.service
        enabled: true
EOF
```

> **`crashkernel=512M`의 의미**
> 부팅 시 OS가 자기 메모리에서 512MB를 떼어내 "crash kernel" 전용으로 예약합니다. 이 메모리는 평소에는 사용 불가하며, panic 발생 시 이 영역에서 별도 미니 커널이 부팅되어 main 커널의 메모리를 덤프 파일로 저장합니다.
> - 너무 작으면 (예: 128M) 덤프 캡처 실패
> - 너무 크면 가용 메모리 손실
> - 권장: 일반 서버 256~512M, 메모리 64GB 이상 시 512M~1G

### 4.4.4 MC 정의 (ap/db/podpool1)

```bash
# AP
cat > 99-ap-kdump.yaml <<'EOF'
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: ap
  name: 99-ap-kdump
spec:
  kernelArguments:
  - crashkernel=512M
  config:
    ignition:
      version: 3.4.0
    systemd:
      units:
      - name: kdump.service
        enabled: true
EOF

# DB
sed 's/ap/db/g' 99-ap-kdump.yaml > 99-db-kdump.yaml

# PodPool1
sed 's/ap/podpool1/g' 99-ap-kdump.yaml > 99-podpool1-kdump.yaml
```

### 4.4.5 적용 및 검증

```bash
oc apply -f 99-master-kdump.yaml
oc apply -f 99-ap-kdump.yaml
oc apply -f 99-db-kdump.yaml
oc apply -f 99-podpool1-kdump.yaml

# MCP rolling reboot 진행 (chrony와 동일)
watch oc get mcp

# 완료 후 검증
oc debug node/master-0 -- chroot /host systemctl status kdump.service
oc debug node/master-0 -- chroot /host cat /proc/cmdline | tr ' ' '\n' | grep crashkernel
# 기대: crashkernel=512M
```

> **kdump MC를 chrony MC와 함께 적용하라**
> 4.3에서 chrony, 4.4에서 kdump를 분리해 설명했지만, **실제로는 두 MC를 함께 적용하여 rolling reboot 횟수를 줄이는 것이 좋습니다**.
> 학습 목적상 분리한 것이며, 실전에서는 chrony+kdump+multipath를 한꺼번에 apply하면 rolling reboot 한 번으로 모든 변경이 적용됩니다.

---

## 4.5 MachineConfig 적용 - multipath

### 4.5.1 multipath의 목적

multipath는 동일 LUN에 다중 경로로 접근할 때 OS 레벨에서 path failover와 부하분산을 처리하는 기능입니다.

본 PoC에서 multipath가 필요한 경우:
- NAS가 NFS가 아닌 iSCSI 기반인 경우
- VM 노드가 SAN 디스크를 사용하는 경우
- 향후 block storage(예: ODF, Cinder) 연동

**본 PoC는 NAS가 NFS 기반이므로 multipath가 필수가 아닙니다.** 다만 향후 확장성과 운영 표준화를 위해 다음 정책으로 적용합니다.

| MCP | multipath 적용 |
|---|---|
| master | 미적용 (필요 시 추가) |
| ap | 적용 (VM 데이터 disk SAN 가능성) |
| db | 적용 (DB VM 디스크 분리 가능성) |
| podpool1 | 미적용 (컨테이너 노드는 일반적으로 불필요) |

### 4.5.2 multipath 설정 파일

```bash
mkdir -p ~/openshift-day2/mc-multipath
cd ~/openshift-day2/mc-multipath
```

```bash
cat > multipath.conf <<'EOF'
defaults {
    user_friendly_names yes
    find_multipaths yes
    path_grouping_policy multibus
    path_checker tur
    no_path_retry queue
}

blacklist {
    # 로컬 디스크 (RHCOS root)는 multipath 대상에서 제외
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^vd[a-z]"
    devnode "^xvd[a-z]"
}

blacklist_exceptions {
    # 필요 시 특정 vendor 추가
}
EOF
```

```bash
MULTIPATH_B64=$(base64 -w0 multipath.conf)
```

### 4.5.3 MC 정의

```bash
# AP MC
cat > 99-ap-multipath.yaml <<EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: ap
  name: 99-ap-multipath
spec:
  kernelArguments:
  - root=/dev/disk/by-label/dm-mpath-root
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
      - path: /etc/multipath.conf
        mode: 0644
        overwrite: true
        contents:
          source: data:text/plain;charset=utf-8;base64,${MULTIPATH_B64}
    systemd:
      units:
      - name: multipathd.service
        enabled: true
EOF

# DB MC (ap와 동일 내용, 라벨만 변경)
sed 's/ap/db/g' 99-ap-multipath.yaml > 99-db-multipath.yaml
```

> **`root=/dev/disk/by-label/dm-mpath-root` 옵션 주의**
> 이 옵션은 root 파일시스템이 multipath device 위에 있을 때 사용합니다. **PoC 환경에서 OS 디스크가 단일 경로라면 이 옵션을 추가하면 부팅이 실패합니다.**
> 본 PoC는 OS 디스크가 단일 경로이고 multipath는 추가 데이터 디스크에만 적용한다고 가정하므로 이 옵션을 제거합니다.

```bash
# 수정 (root kernel arg 제거)
cat > 99-ap-multipath.yaml <<EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: ap
  name: 99-ap-multipath
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
      - path: /etc/multipath.conf
        mode: 0644
        overwrite: true
        contents:
          source: data:text/plain;charset=utf-8;base64,${MULTIPATH_B64}
    systemd:
      units:
      - name: multipathd.service
        enabled: true
EOF

sed 's/ap/db/g' 99-ap-multipath.yaml > 99-db-multipath.yaml
```

### 4.5.4 적용 및 검증

```bash
oc apply -f 99-ap-multipath.yaml
oc apply -f 99-db-multipath.yaml

# 진행 모니터링
watch oc get mcp

# 검증
oc debug node/ap-worker-0 -- chroot /host systemctl status multipathd
oc debug node/ap-worker-0 -- chroot /host multipath -ll
# 단일 디스크 환경에서는 출력이 없거나 멤버 디스크가 없음 (정상)
```

---

## 4.6 MC 적용 통합 전략

### 4.6.1 권장 적용 순서

PoC에서 실제로 적용할 때는 학습 목적이 아니라면 다음 순서로 한 번에 진행합니다.

```bash
# 1. MCP 먼저 생성 (rolling reboot 없음)
oc apply -f ~/openshift-day2/mcp/

# 2. 노드 라벨링 (MCP 편입 자동, rolling reboot 없음)
# (이미 4.2.2에서 수행)

# 3. 모든 MC를 한 번에 적용 (rolling reboot 1회로 끝)
oc apply -f ~/openshift-day2/mc-chrony/
oc apply -f ~/openshift-day2/mc-kdump/
oc apply -f ~/openshift-day2/mc-multipath/

# 4. MCP 완료 대기
oc wait --for=condition=Updated --timeout=120m mcp/master
oc wait --for=condition=Updated --timeout=60m mcp/ap
oc wait --for=condition=Updated --timeout=60m mcp/db
oc wait --for=condition=Updated --timeout=60m mcp/podpool1
```

### 4.6.2 적용 순서가 중요한 이유

**1. MCP가 없는 상태에서 MC를 만들면**
- MC의 label selector(`machineconfiguration.openshift.io/role: ap`)가 매칭되는 MCP가 없음
- MC는 생성되지만 어떤 노드에도 적용되지 않음
- 이후 MCP를 만들면 그제서야 적용됨

**2. MCP만 만들고 MC 없이 노드 라벨링**
- 라벨이 매칭되는 노드들이 새 MCP로 편입
- 새 MCP의 rendered config는 기본 worker 설정과 동일
- rolling reboot 발생하지 않음

**3. MC를 여러 개 차례로 적용**
- 각 apply마다 별도 rendered config 생성
- 매번 rolling reboot 트리거
- 비효율적 (시간 낭비)

**4. MC를 한 번에 apply**
- 모든 MC가 병합되어 단일 rendered config 생성
- rolling reboot 1회로 완료

> **MCP 단위로 rolling reboot은 병렬 진행**
> ap, db, podpool1 MCP는 서로 독립이므로 동시에 rolling reboot이 가능합니다. 단, **master는 단독으로 진행**되며 etcd 안정성을 위해 한 대씩 신중하게 처리됩니다.

### 4.6.3 적용 완료 검증

```bash
# 모든 MCP가 Updated인지 확인
oc get mcp -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Updated")].status}{"\n"}{end}'
```

기대:
```
master      True
ap          True
db          True
podpool1    True
worker      True
```

각 노드에서 적용 확인:

```bash
# chrony 확인
oc debug node/podpool1-worker-0 -- chroot /host chronyc sources

# kdump 확인
oc debug node/podpool1-worker-0 -- chroot /host systemctl is-active kdump

# multipath 확인 (ap/db만)
oc debug node/ap-worker-0 -- chroot /host systemctl is-active multipathd
```

---

## 4.7 인증서 관리

### 4.7.1 OpenShift 인증서 구조

OpenShift는 여러 종류의 인증서를 사용합니다.

| 인증서 | 용도 | 발급자 | 교체 권한 |
|---|---|---|---|
| API 서버 (api) | 외부 API 접근 | 클러스터 내부 CA (기본) | 사용자 교체 가능 |
| API 서버 내부 (api-int) | 노드 → API 통신 | 클러스터 내부 CA | 자동, 교체 불가 |
| Ingress 와일드카드 (`*.apps`) | Route TLS | 클러스터 내부 CA (기본) | 사용자 교체 가능 |
| Etcd peer | etcd 노드 간 통신 | 클러스터 내부 CA | 자동, 교체 불가 |
| Kubelet | API 서버 인증 | 클러스터 CA (CSR로 발급) | 자동 갱신 |
| Service Account | Pod 인증 | 클러스터 SA signer | 자동 |

**사용자가 교체 가능하고 권장되는 두 가지:**
1. API 서버 외부 인증서 (`api.<cluster>.<domain>`)
2. Ingress 와일드카드 인증서 (`*.apps.<cluster>.<domain>`)

### 4.7.2 인증서 교체 시점

| 시점 | 권장 작업 |
|---|---|
| PoC 초기 | 자체 서명 그대로 사용 |
| 외부 사용자 접근 시작 전 | 사내 CA 또는 신뢰 CA로 교체 |
| 운영 전환 | 모든 외부 노출 인증서 교체 |

PoC 단계에서는 자체 서명을 유지해도 무방하지만, 본 문서는 사내 CA 발급 인증서로 교체하는 절차를 다룹니다.

### 4.7.3 사내 CA 인증서 준비

운영자가 사내 CA로부터 발급받아야 하는 인증서:

**API 서버 인증서:**
- CN: `api.ocp1.example.com`
- SAN: `api.ocp1.example.com`

**Ingress 인증서 (와일드카드):**
- CN: `*.apps.ocp1.example.com`
- SAN: `*.apps.ocp1.example.com`

> **Service Ingress용 인증서 별도 필요**
> 본 PoC는 서비스용 Ingress 도메인(`*.svcapps.ocp1.example.com`)을 분리했으므로, 이를 위한 인증서도 별도로 준비해야 합니다.
> - CN: `*.svcapps.ocp1.example.com`
> - SAN: `*.svcapps.ocp1.example.com`

### 4.7.4 Ingress 인증서 교체

```bash
# Secret 생성 (default IngressController용)
oc create secret tls custom-default-ingress-cert \
  --cert=apps-ocp1.crt \
  --key=apps-ocp1.key \
  -n openshift-ingress

# IngressController 패치
oc patch ingresscontroller default \
  -n openshift-ingress-operator \
  --type=merge \
  --patch='{"spec":{"defaultCertificate":{"name":"custom-default-ingress-cert"}}}'

# 변경 확인 (router pod 재시작)
oc get pods -n openshift-ingress
```

### 4.7.5 API 서버 인증서 교체

```bash
# Secret 생성
oc create secret tls custom-api-cert \
  --cert=api-ocp1.crt \
  --key=api-ocp1.key \
  -n openshift-config

# APIServer 리소스 패치
oc patch apiserver cluster --type=merge -p '{
  "spec": {
    "servingCerts": {
      "namedCertificates": [{
        "names": ["api.ocp1.example.com"],
        "servingCertificate": {
          "name": "custom-api-cert"
        }
      }]
    }
  }
}'

# 적용까지 5~10분 소요 (kube-apiserver operator가 처리)
oc get co kube-apiserver
```

### 4.7.6 인증서 만료 모니터링

```bash
# 모든 인증서 만료 일자 조회
oc get secret -A -o json | jq -r '
  .items[]
  | select(.data."tls.crt" != null)
  | "\(.metadata.namespace)\t\(.metadata.name)\t\(.data."tls.crt" | @base64d | split("\n")[1:-1] | join("\n") | "echo \"\(.)\" | openssl x509 -noout -enddate")
'

# 또는 cert-manager Operator로 자동 갱신 (4.20 권장)
```

> **Mirror Registry CA 만료**
> Part 2에서 mirror-registry가 자체 생성한 CA는 보통 10년 유효합니다. 단, 운영 환경에서 Mirror Registry를 분리하면 사내 CA로 발급 받은 인증서를 사용하게 되며, 그 인증서가 만료되면 모든 노드의 이미지 pull이 멈춥니다. 만료 60일 전 알림 설정을 권장합니다.

---

## 4.8 etcd 백업 정책

### 4.8.1 etcd 백업의 중요성

etcd는 클러스터의 모든 상태를 저장하는 분산 키-값 저장소입니다. etcd 손실 = 클러스터 손실입니다.

복구 가능한 시나리오:
- 1대 master 손실: 나머지 2대가 quorum 유지, 손실 master 교체 가능
- 2대 master 손실: quorum 깨짐, etcd 복구 필요
- 3대 master 손실: 백업에서 etcd 완전 복구 필요

### 4.8.2 백업 절차

OpenShift는 `cluster-backup.sh` 스크립트를 제공합니다.

```bash
# master 노드에 SSH 접속
ssh -i ~/.ssh/ocp_id_ed25519 core@master-0

# 백업 실행
sudo -E /usr/local/bin/cluster-backup.sh /home/core/backup
```

기대 출력:
```
Certificate /etc/kubernetes/static-pod-resources/kube-apiserver-certs/...
found latest kube-apiserver: ...
found latest kube-controller-manager: ...
found latest kube-scheduler: ...
found latest etcd: ...
{"level":"info","ts":"2026-05-15T...","caller":"snapshot/v3_snapshot.go:65","msg":"created temporary db file"}
{"level":"info","ts":"...","caller":"snapshot/v3_snapshot.go:73","msg":"fetching snapshot"}
snapshot db and kube resources are successfully saved to /home/core/backup
```

산출물:
```
/home/core/backup/
├── snapshot_YYYY-MM-DD_HHMMSS.db          # etcd 스냅샷
└── static_kuberesources_YYYY-MM-DD_HHMMSS.tar.gz  # static pod 매니페스트
```

### 4.8.3 백업 자동화

PoC에서는 매일 또는 주 1회 자동 백업을 권장합니다.

**옵션 1: master 노드의 cron**

```bash
# master-0에서
sudo crontab -e
```

```cron
# 매일 새벽 2시 백업
0 2 * * * /usr/local/bin/cluster-backup.sh /home/core/backup-$(date +\%Y\%m\%d) 2>&1 | logger -t etcd-backup
```

**옵션 2: OpenShift CronJob (4.20 신기능)**

OpenShift 4.20에서는 etcd 자동 백업이 클러스터 차원에서 지원됩니다.

```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: EtcdBackup
metadata:
  name: etcd-daily-backup
spec:
  pvcName: etcd-backup-pvc  # NAS PVC 권장
  schedule: "0 2 * * *"
  timeZone: "Asia/Seoul"
  retentionPolicy:
    retentionType: RetentionNumber
    retentionNumber:
      maxNumberOfBackups: 7
```

### 4.8.4 백업 보관 위치

PoC에서는 NAS에 백업을 저장합니다.

```bash
# master 노드의 /home/core/backup → Bastion → NAS

# 옵션 A: master에서 직접 NAS mount
sudo mount -t nfs 192.168.10.70:/exports/etcd-backup /mnt/backup

# 옵션 B: Bastion이 master에서 SCP로 가져온 후 NAS 보관
scp -r core@master-0:/home/core/backup/ /mnt/nas-backup/
```

**보관 정책 (PoC 권장):**
- 일일 백업: 7일 보관
- 주간 백업: 4주 보관
- 월간 백업: 6개월 보관

### 4.8.5 백업 검증

백업 파일이 단순히 존재하는 것만으로는 부족합니다. 복원 가능한지 정기적으로 검증해야 합니다.

```bash
# 백업 파일 무결성 검증
file /home/core/backup/snapshot_*.db
# 기대: SQLite 3.x database, ...

# etcdctl로 백업 검증 (master 노드)
sudo -i
ETCDCTL_API=3 etcdctl snapshot status \
  /home/core/backup/snapshot_*.db \
  --write-out=table
```

기대 출력:
```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| ed3xxxxx |   123456 |       7890 |     150 MB |
+----------+----------+------------+------------+
```

복원 절차는 별도의 disaster recovery 가이드에서 다루는 것이 일반적이며, PoC에서는 백업 생성과 검증까지를 범위로 합니다.

---

## 4.9 노드 디스크 설계

### 4.9.1 RHCOS 디스크 레이아웃

RHCOS는 기본적으로 단일 디스크에 다음 구조로 설치됩니다.

```
/dev/sda (또는 /dev/vda)
├── /boot                       (~384 MiB)
├── /boot/efi                   (~127 MiB)
├── / (root, xfs)               (나머지)
│   └── /var (rootfs 내 디렉토리)
│       ├── /var/lib/containers (이미지, 컨테이너 layer)
│       ├── /var/lib/kubelet    (Pod 데이터)
│       ├── /var/log
│       ├── /var/lib/etcd       (master만)
│       └── /var/crash          (kdump)
```

기본 설치는 단일 파티션이므로 `/var`가 root 파일시스템과 같은 디스크/파티션입니다.

### 4.9.2 디스크 분리의 필요성

| 분리 대상 | 이유 |
|---|---|
| `/var` 분리 | 컨테이너 이미지/로그 폭증으로 root가 가득 차 시스템 멈추는 것 방지 |
| `/var/lib/containers` 분리 | 컨테이너 이미지를 별도 디스크에 두어 root 안정성 확보 |
| `/var/lib/etcd` 분리 (master) | etcd I/O를 root와 분리하여 latency 보장 |

### 4.9.3 PoC에서의 권장 디스크 설계

**Master 노드:**

| 디스크 | 크기 | 마운트 | 용도 |
|---|---|---|---|
| /dev/sda | 120 GB | / (xfs) | OS + 기본 /var |
| /dev/sdb (선택) | 50 GB | /var/lib/etcd | etcd 전용 (권장) |

**AP/DB Worker 노드:**

| 디스크 | 크기 | 마운트 | 용도 |
|---|---|---|---|
| /dev/sda | 100 GB | / (xfs) | OS |
| /dev/sdb | 200 GB | /var/lib/kubelet/pods | VM ephemeral 디스크 (필요 시) |
| /dev/sdc | 100 GB | /var/lib/containers | 컨테이너 이미지 |

**PodPool1 Worker 노드:**

| 디스크 | 크기 | 마운트 | 용도 |
|---|---|---|---|
| /dev/sda | 100 GB | / (xfs) | OS |
| /dev/sdb | 200 GB | /var/lib/containers | 컨테이너 이미지 (Tekton 빌드 캐시 등 누적) |
| /dev/sdc | 100 GB | /var/log | 컨테이너 로그 |

> **PoC 시작 단계에서는 단순화 권장**
> 위 권장은 운영 환경 기준입니다. PoC 초기에는 단일 디스크 200GB로 시작하고, 디스크 사용량 모니터링 후 필요한 노드에만 추가 디스크를 마운트하는 방식이 학습 부담을 줄입니다.

### 4.9.4 추가 디스크 마운트 (MachineConfig)

추가 디스크를 마운트하는 MachineConfig 예시:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: podpool1
  name: 99-podpool1-var-containers
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      disks:
      - device: /dev/sdb
        wipeTable: false
        partitions:
        - label: var-containers
          number: 1
          sizeMiB: 0    # 디스크 전체 사용
          startMiB: 0
      filesystems:
      - device: /dev/disk/by-partlabel/var-containers
        format: xfs
        path: /var/lib/containers
        label: var-containers
    systemd:
      units:
      - name: var-lib-containers.mount
        enabled: true
        contents: |
          [Unit]
          Before=local-fs.target
          Requires=systemd-fsck@dev-disk-by\x2dpartlabel-var\x2dcontainers.service
          After=systemd-fsck@dev-disk-by\x2dpartlabel-var\x2dcontainers.service

          [Mount]
          Where=/var/lib/containers
          What=/dev/disk/by-partlabel/var-containers
          Type=xfs

          [Install]
          WantedBy=local-fs.target
```

> **이 MC를 적용하면 노드 데이터 손실 위험**
> 추가 디스크 마운트 MC는 신중하게 적용해야 합니다. `wipeTable: true`로 잘못 설정하거나, 이미 사용 중인 디스크를 잡으면 데이터 손실이 발생합니다.
> 본 PoC에서는 새 디스크가 추가된 상태에서만 이 MC를 적용하시기를 권장합니다.

### 4.9.5 디스크 사용량 모니터링

```bash
# 모든 노드의 disk 사용량 일괄 확인
for node in $(oc get nodes -o name); do
  echo "=== $node ==="
  oc debug $node -- chroot /host df -h / /var 2>/dev/null
done

# 컨테이너 이미지 저장 사용량
for node in $(oc get nodes -o name); do
  echo "=== $node ==="
  oc debug $node -- chroot /host du -sh /var/lib/containers 2>/dev/null
done
```

`/var` 사용량이 80%를 넘어가면 다음 조치를 검토합니다.
- 사용하지 않는 이미지 정리 (`crictl rmi --prune`)
- 로그 회전 정책 검토
- 추가 디스크 마운트

---

## 4.10 Day-2 운영 체크리스트

본 Part 4에서 다룬 작업의 완료 체크리스트입니다.

| 항목 | 확인 명령 | 기대 결과 |
|---|---|---|
| MCP 분리 완료 | `oc get mcp` | ap, db, podpool1, master, worker(0대) |
| 노드 라벨 적용 | `oc get nodes` | ROLES에 ap/db/podpool1 표시 |
| chrony 적용 | `oc debug node/<n> -- chronyc sources` | Reference ID = Bastion(192.168.10.10) |
| kdump 활성화 | `oc debug node/<n> -- systemctl is-active kdump` | active |
| crashkernel 예약 | `oc debug node/<n> -- cat /proc/cmdline \| grep crashkernel` | crashkernel=512M |
| multipath (ap/db) | `oc debug node/<n> -- systemctl is-active multipathd` | active |
| Ingress 인증서 교체 | `oc get ingresscontroller/default -n openshift-ingress-operator -o jsonpath='{.spec.defaultCertificate.name}'` | custom-default-ingress-cert |
| etcd 백업 동작 | `ls /mnt/nas-backup/` | snapshot_YYYY-MM-DD_*.db |
| 디스크 사용량 | `df -h /var` | < 80% 사용 |

---

## 4.11 Part 4 학습 점검

다음 질문에 답할 수 있다면 Part 4를 충분히 학습한 것입니다.

1. MachineConfigPool과 MachineConfig의 관계는? 노드는 어떤 MCP에 속하는지 어떻게 결정되는가?
2. MCP의 `machineConfigSelector`에서 `worker`를 빼고 `ap`만 두면 어떤 문제가 발생하는가?
3. MCP만 새로 생성하고 추가 MC를 만들지 않으면 rolling reboot이 발생하는가?
4. 본 PoC에서 ap와 db를 별도 MCP로 분리한 다섯 가지 근거를 설명하시오.
5. worker 라벨 유지 vs 제거의 trade-off는 무엇인가? 본 PoC가 유지를 권장하는 이유는?
6. MC를 차례로 적용할 때와 한 번에 적용할 때의 차이는?
7. chrony MC에서 Bastion IP를 직접 명시하는 이유는?
8. `crashkernel=512M`의 의미는? 너무 작거나 크면 어떻게 되는가?
9. multipath MC에서 `root=/dev/disk/by-label/dm-mpath-root` 옵션을 함부로 추가하면 안 되는 이유는?
10. Ingress 인증서를 교체하면 어떤 Pod이 재시작되는가?
11. etcd 백업이 단순히 존재하는 것만으로 부족한 이유는? 검증은 어떻게 하는가?
12. PodPool1 노드에서 `/var`가 가득 차면 어떤 문제가 발생하는가? 예방 방법은?

---

*Part 4 끝. 다음은 Part 5 (Operator 설치 및 OpenShift Virtualization) 입니다.*
