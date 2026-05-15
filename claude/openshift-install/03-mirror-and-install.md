# Part 3. 폐쇄망 미러링 및 OpenShift 설치

> **이 파트의 목적**
> 폐쇄망(에어갭) 환경에서 OpenShift를 설치하는 핵심 절차를 다룹니다. 핵심은 두 단계입니다.
> 첫째, 외부망 환경에서 필요한 모든 컨테이너 이미지를 파일(tar 아카이브)로 미러링합니다.
> 둘째, 그 파일을 폐쇄망으로 물리적으로 이동시킨 후, Bastion의 Mirror Registry(Quay)에 업로드하고, 그 미러 정보를 install-config.yaml에 반영하여 OpenShift를 UPI 방식으로 설치합니다.
>
> 이 파트를 학습한 후 학습자는 oc-mirror v2의 동작 원리, 미러링과 설치의 의존 관계, UPI 설치의 단계별 흐름을 설명할 수 있어야 합니다.

---

## 3.1 oc-mirror v2 개요

### 3.1.1 oc-mirror의 역할

`oc-mirror`는 OpenShift 폐쇄망 설치를 위한 공식 미러링 도구입니다. 다음을 한 번에 처리합니다.

- OpenShift release image 미러링
- Operator catalog와 카탈로그 내 Operator 이미지 미러링
- 사용자 지정 추가 이미지 미러링 (JBoss, UBI 등)
- 미러 정보를 담은 OpenShift 리소스 자동 생성 (IDMS, ITMS, CatalogSource)

### 3.1.2 v1과 v2의 차이

oc-mirror에는 v1과 v2가 있으며, **OpenShift 4.20에서는 v2 사용을 권장**합니다.

| 항목 | v1 (legacy) | v2 (권장) |
|---|---|---|
| 명령어 플래그 | 기본값 | `--v2` 명시 필요 |
| 미러 정책 리소스 | ImageContentSourcePolicy (ICSP) | ImageDigestMirrorSet (IDMS) + ImageTagMirrorSet (ITMS) |
| 산출물 디렉토리 | `oc-mirror-workspace/` | `working-dir/`, `cluster-resources/` |
| 증분 미러링 | metadata 기반 (복잡) | 단순화, 명시적 |
| 다중 카탈로그 처리 | 제한적 | 개선됨 |
| Helm chart 미러링 | 지원 | 지원 (개선됨) |

> **왜 v2를 사용해야 하는가**
> 1. **OpenShift 4.14부터 ImageContentSourcePolicy(ICSP)가 deprecated** — v1이 생성하는 ICSP는 향후 OpenShift 버전에서 동작하지 않을 수 있습니다.
> 2. **IDMS/ITMS는 더 표현력이 풍부** — Tag 기반 미러와 Digest 기반 미러를 명확히 분리합니다.
> 3. **산출물 구조가 명확** — `cluster-resources/` 디렉토리에 OpenShift에 적용할 매니페스트가 한곳에 모입니다.
> 4. **OpenShift 4.20 공식 문서가 v2 기준** — 새로 학습하는 입장에서 일관성 있는 자료를 참조 가능합니다.

### 3.1.3 2단계 미러링 흐름

폐쇄망 환경의 미러링은 두 단계로 진행됩니다.

```
[1단계] 인터넷 가능 환경 (외부망 워크스테이션)
   │
   │  oc-mirror -c imageset-config.yaml \
   │    file:///path/to/output --v2
   │
   ▼
   tar 아카이브 생성
   - mirror_seq1_000000.tar
   - mirror_seq1_000001.tar (분할되었다면)
   - working-dir/  (메타데이터)
   │
   │  USB / 외장하드 / DVD 등으로 물리 이동
   │
   ▼
[2단계] 폐쇄망 환경 (Bastion)
   │
   │  oc-mirror -c imageset-config.yaml \
   │    --from=/path/to/tar-archives \
   │    docker://mirror.ocp1.example.com:8443 --v2
   │
   ▼
   Mirror Registry에 이미지 업로드
   cluster-resources/ 생성
     - idms-oc-mirror.yaml
     - itms-oc-mirror.yaml
     - cs-redhat-operator-index.yaml (CatalogSource)
     - ...
```

### 3.1.4 학습 포인트

학습 시 반드시 이해해야 할 핵심 개념입니다.

**1. ImageSet 설정 파일이 모든 것의 시작**

`imageset-config.yaml`이라는 단일 YAML 파일이 무엇을 미러링할지 정의합니다. 이 파일이 1단계와 2단계 모두에서 동일하게 사용됩니다. 즉, **1단계에서 받은 이미지 목록과 2단계에서 업로드하는 이미지 목록이 같음을 보장**합니다.

**2. tar 아카이브는 OCI 형식 이미지의 컨테이너**

oc-mirror가 생성하는 tar는 단순한 압축 파일이 아니라 OCI(Open Container Initiative) 형식으로 정리된 이미지 컨테이너입니다. 이 안에는 release 이미지, Operator 이미지, signature, layer blob이 모두 들어있습니다.

**3. cluster-resources는 OpenShift에 적용할 매니페스트**

2단계 완료 후 생성되는 `cluster-resources/` 디렉토리는 단순한 로그가 아니라, **클러스터가 미러를 사용하도록 만드는 핵심 매니페스트들**입니다. 이를 `oc apply -f`로 적용해야 클러스터가 미러를 인식합니다.

---

## 3.2 ImageSet 설계

### 3.2.1 imageset-config.yaml 개요

ImageSet 설정 파일은 미러링 대상을 선언적으로 정의합니다. 전체 구조는 다음과 같습니다.

```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  platform:        # OpenShift release image
    channels:
    - name: stable-4.20
      minVersion: 4.20.0
      maxVersion: 4.20.0
  operators:       # Operator catalogs
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:
    - name: kubernetes-nmstate-operator
    - name: kubevirt-hyperconverged
    # ...
  additionalImages: # 추가 이미지 (JBoss, UBI 등)
  - name: registry.redhat.io/ubi9/ubi:latest
  # ...
```

### 3.2.2 본 PoC의 imageset-config.yaml (전체)

다음은 본 PoC에서 사용할 완전한 imageset-config.yaml 예시입니다.

```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration

mirror:
  # ==========================================
  # OpenShift Container Platform release image
  # ==========================================
  platform:
    architectures:
    - amd64
    channels:
    - name: stable-4.20
      type: ocp
      minVersion: 4.20.0
      maxVersion: 4.20.0    # PoC 시점의 z-stream 버전 명시
    graph: true

  # ==========================================
  # Red Hat Operators
  # ==========================================
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:

    # 네트워크 구성 (VM bridge 등)
    - name: kubernetes-nmstate-operator
      channels:
      - name: stable

    # OpenShift Virtualization
    - name: kubevirt-hyperconverged
      channels:
      - name: stable

    # Migration Toolkit for Virtualization
    - name: mtv-operator
      channels:
      - name: release-v2.7    # 4.20 호환 버전 확인

    # OpenShift Pipelines (Tekton)
    - name: openshift-pipelines-operator-rh
      channels:
      - name: latest

    # OpenShift GitOps
    - name: openshift-gitops-operator
      channels:
      - name: latest

    # Service Mesh
    - name: servicemeshoperator
      channels:
      - name: stable

    # Node Health Check (FAR 의존)
    - name: node-healthcheck-operator
      channels:
      - name: stable

    # Fence Agents Remediation
    - name: fence-agents-remediation
      channels:
      - name: stable

    # (선택) Cluster Logging - Loki
    - name: cluster-logging
      channels:
      - name: stable-6.0
    - name: loki-operator
      channels:
      - name: stable-6.0

  # ==========================================
  # Additional images
  # ==========================================
  additionalImages:
  # UBI base
  - name: registry.redhat.io/ubi9/ubi:latest
  - name: registry.redhat.io/ubi9/ubi-minimal:latest

  # JBoss EAP base images
  - name: registry.redhat.io/jboss-eap-7/eap74-openjdk17-openshift-rhel8:latest
  - name: registry.redhat.io/jboss-eap-8/eap8-openjdk17-builder-openshift-rhel8:latest
  - name: registry.redhat.io/jboss-eap-8/eap8-openjdk17-runtime-openshift-rhel8:latest

  # (선택) 테스트 워크로드 이미지
  - name: registry.redhat.io/rhel9/httpd-24:latest
```

### 3.2.3 설정 요소 해설

**`platform.channels`**

```yaml
platform:
  channels:
  - name: stable-4.20
    type: ocp
    minVersion: 4.20.0
    maxVersion: 4.20.0
  graph: true
```

- `name: stable-4.20`: 채널 이름. stable-4.20, fast-4.20, candidate-4.20 중 stable이 안정성 우선.
- `minVersion / maxVersion`: 같은 값으로 두면 해당 단일 버전만 미러링. 범위를 두면 그 사이 모든 z-stream을 미러링하므로 디스크 사용량이 크게 증가.
- `graph: true`: 클러스터 업그레이드를 위한 OSUS(OpenShift Update Service) graph 데이터 포함. 추후 업그레이드 검증 시 필요.

**`operators.catalog`**

```yaml
operators:
- catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
  packages:
  - name: kubernetes-nmstate-operator
```

- `catalog`: Operator 카탈로그 인덱스 이미지. Red Hat은 4가지 카탈로그를 제공.
  - `redhat-operator-index`: Red Hat 인증 Operator
  - `certified-operator-index`: 파트너 인증 Operator
  - `community-operator-index`: 커뮤니티 Operator
  - `redhat-marketplace-index`: 마켓플레이스 Operator
- `packages`: 미러링할 Operator만 선별. 카탈로그 전체를 받으면 수백 GB가 되므로 반드시 선별.
- `channels`: 명시하지 않으면 default channel만 가져옴. 특정 채널을 원하면 명시.

**`additionalImages`**

미러 레지스트리에 그대로 복사할 추가 이미지. Operator로 관리되지 않는 base image나 사용자 워크로드 이미지를 여기에 명시.

> **흔한 실수: maxVersion을 빈 값으로 두기**
> `maxVersion`을 명시하지 않거나 `maxVersion: ""`로 두면 채널 내 모든 버전이 미러링됩니다. 4.20 채널 전체를 미러링하면 수백 GB가 추가됩니다. PoC는 단일 버전만 충분합니다.

### 3.2.4 pull-secret 준비

Red Hat 레지스트리(`registry.redhat.io`)와 Mirror Registry(Quay)에 인증하기 위한 pull-secret을 준비해야 합니다.

**1단계: Red Hat pull-secret 다운로드**

외부망 워크스테이션에서:

```bash
# https://console.redhat.com/openshift/install/pull-secret
# 위 페이지에서 pull-secret 다운로드 후 저장
mkdir -p ~/.docker
mv ~/Downloads/pull-secret.json ~/.docker/config.json
```

**2단계: Mirror Registry 인증 정보 추가 (폐쇄망 환경)**

pull-secret에는 Red Hat 인증 정보만 있습니다. 폐쇄망 Bastion의 Mirror Registry에도 인증해야 하므로 정보를 추가합니다.

```bash
# Mirror Registry 인증 정보를 base64로 인코딩
echo -n "admin:RedHat123!" | base64
# 결과: YWRtaW46UmVkSGF0MTIzIQ==

# pull-secret 편집
vi pull-secret.json
```

수정 후 모습:

```json
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "<Red Hat 인증>",
      "email": "you@example.com"
    },
    "quay.io": {
      "auth": "<Red Hat 인증>",
      "email": "you@example.com"
    },
    "registry.connect.redhat.com": {
      "auth": "<Red Hat 인증>"
    },
    "registry.redhat.io": {
      "auth": "<Red Hat 인증>"
    },
    "mirror.ocp1.example.com:8443": {
      "auth": "YWRtaW46UmVkSGF0MTIzIQ==",
      "email": "admin@ocp1.example.com"
    }
  }
}
```
jq -s '.[0] * .[1]' pull-secret.txt local-secret.json > merged-pull-secret.json


> **두 pull-secret이 필요한 이유**
> - 1단계(외부망 미러링): Red Hat 레지스트리에서 pull할 때 필요 → Red Hat 인증만 있어도 충분
> - 2단계(폐쇄망 업로드): Mirror Registry에 push할 때 필요 → Mirror Registry 인증 추가 필요
> - install-config.yaml: 클러스터가 Mirror Registry에서 pull할 때 필요 → Mirror Registry 인증 필수
>
> 보통은 두 환경에서 사용할 통합 pull-secret을 하나 만들어 두는 것이 편합니다.

---

## 3.3 [1단계] 외부망에서 미러링

### 3.3.1 환경 준비

외부망 워크스테이션 요구사항:

| 항목 | 최소 | 권장 |
|---|---|---|
| OS | RHEL 9.x, Fedora, Ubuntu 22+ | RHEL 9.4+ |
| CPU | 4 core | 8 core |
| Memory | 16 GB | 32 GB |
| Disk | 500 GB | 1 TB (미러링 산출물 저장) |
| 인터넷 | registry.redhat.io 접근 | 안정적 회선 (수십 GB 다운로드) |

**oc-mirror 도구 설치**

```bash
# Red Hat 공식 다운로드 페이지에서 oc-mirror 다운로드
# https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.20.x/

cd /tmp
tar -xzf oc-mirror.tar.gz
sudo install -m 0755 oc-mirror /usr/local/bin/

# 실행 권한 확인
oc-mirror version
# 기대: WARNING: This version of oc-mirror is going to be deprecated in a future release...
#       Client Version: ...

# 도구가 호출되는지 확인
oc-mirror help
```

> **oc-mirror v2 사용 시 항상 `--v2` 플래그**
> v2 기능을 사용하려면 명령마다 `--v2` 플래그를 명시해야 합니다. 빠뜨리면 v1 동작으로 진행되어 산출물이 다릅니다.

### 3.3.2 작업 디렉토리 구성

```bash
# 미러링 작업 루트
mkdir -p ~/oc-mirror-work
cd ~/oc-mirror-work

# 디렉토리 구조 준비
mkdir -p output    # tar 아카이브 출력 위치

# pull-secret 배치
mkdir -p ~/.docker
cp /path/to/pull-secret.json ~/.docker/config.json
chmod 600 ~/.docker/config.json
```

### 3.3.3 imageset-config.yaml 배치

위에서 작성한 imageset-config.yaml을 작업 디렉토리에 둡니다.

```bash
cat > ~/oc-mirror-work/imageset-config.yaml <<'EOF'
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  platform:
    architectures:
    - amd64
    channels:
    - name: stable-4.20
      type: ocp
      minVersion: 4.20.0
      maxVersion: 4.20.0
    graph: true
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:
    - name: kubernetes-nmstate-operator
    - name: kubevirt-hyperconverged
    - name: mtv-operator
    - name: openshift-pipelines-operator-rh
    - name: openshift-gitops-operator
    - name: servicemeshoperator
    - name: node-healthcheck-operator
    - name: fence-agents-remediation
  additionalImages:
  - name: registry.redhat.io/ubi9/ubi:latest
  - name: registry.redhat.io/ubi9/ubi-minimal:latest
  - name: registry.redhat.io/jboss-eap-7/eap74-openjdk17-openshift-rhel8:latest
  - name: registry.redhat.io/jboss-eap-8/eap8-openjdk17-builder-openshift-rhel8:latest
  - name: registry.redhat.io/jboss-eap-8/eap8-openjdk17-runtime-openshift-rhel8:latest
EOF
```

### 3.3.4 미러링 사전 검증 (dry-run)

본 미러링 전에 어떤 이미지가 미러링될지 예측해보는 단계입니다.

```bash
cd ~/oc-mirror-work

# dry-run으로 미러링 대상 미리 확인 (실제 다운로드 안 함)
oc-mirror --config=imageset-config.yaml \
  file://output \
  --v2 \
  --dry-run
```

산출물:
```
output/working-dir/dry-run/
├── mapping.txt        # 모든 이미지의 source → destination 매핑
└── missing.txt        # 가져올 수 없는 이미지 목록 (있다면)
```

검토 포인트:
- `mapping.txt` 행 수 = 미러링될 이미지 개수
- `missing.txt`가 비어있어야 함 (있다면 권한 문제 또는 잘못된 패키지 이름)

```bash
wc -l output/working-dir/dry-run/mapping.txt
# 예: 850 (대략적인 이미지 수)

cat output/working-dir/dry-run/missing.txt
# 비어있어야 함
```

### 3.3.5 실제 미러링 실행

```bash
cd ~/oc-mirror-work

# 본 미러링 실행
oc-mirror --config=imageset-config.yaml \
  file://output \
  --v2

# 옵션 설명:
# --config:     imageset-config.yaml 경로
# file://output: 산출물을 output/ 디렉토리에 tar로 생성
# --v2:         v2 모드 사용
```

미러링이 진행되며 다음과 같은 로그가 출력됩니다.

```
2026/05/15 10:00:00 [INFO]   : 🔂 Mirror progress: 1/850 (0.12%)
2026/05/15 10:00:30 [INFO]   : 🔂 Mirror progress: 50/850 (5.88%)
...
2026/05/15 14:30:00 [INFO]   : ✅ Mirroring completed successfully
2026/05/15 14:30:01 [INFO]   : 📦 Archive created: output/mirror_000001.tar
```

소요 시간 추정 (네트워크 속도별):
- 100Mbps: 약 8~12시간
- 1Gbps: 약 1~3시간
- 10Gbps: 약 30분~1시간

### 3.3.6 산출물 확인

```bash
ls -lh output/
```

기대 출력:
```
total 350G
-rw-r--r-- 1 user user 100G mirror_000001.tar
-rw-r--r-- 1 user user 100G mirror_000002.tar
-rw-r--r-- 1 user user  50G mirror_000003.tar
drwxr-xr-x 5 user user 4.0K working-dir/
```

> **tar 파일이 여러 개로 분할되는 이유**
> oc-mirror는 기본적으로 단일 tar 파일이 너무 커지지 않도록 자동 분할합니다. (기본 100GB 단위)
> 명시적으로 크기를 조정하려면 `--archiveSize=50` (단위: GB) 옵션을 사용합니다.

`working-dir/`에는 미러링 메타데이터가 있습니다. **이 디렉토리도 2단계에 필요하므로 함께 이동해야 합니다.**

```bash
# 전체 산출물 크기 확인
du -sh output/
# 예: 350G output/
```

### 3.3.7 폐쇄망으로 물리 이동

```bash
# 외장 디스크 마운트 (예: /mnt/external)
sudo mount /dev/sdX1 /mnt/external

# 전체 output 디렉토리를 외장 디스크로 복사
rsync -av --progress output/ /mnt/external/oc-mirror-output/

# 또는 단일 tar로 묶기 (보안 매체 정책상)
tar -cf /mnt/external/oc-mirror-output.tar output/

# 무결성 검증을 위한 해시 생성
cd /mnt/external
sha256sum oc-mirror-output.tar > oc-mirror-output.sha256

# 안전하게 unmount
sync
sudo umount /mnt/external
```

폐쇄망 매체 운영 절차에 따라 매체를 폐쇄망 영역으로 반입합니다.

---

## 3.4 [2단계] 폐쇄망에서 Mirror Registry로 업로드

### 3.4.1 사전 확인

폐쇄망 Bastion에서 다음이 준비되어 있어야 합니다 (Part 2에서 구성).

```bash
# Mirror Registry 동작 확인
podman login mirror.ocp1.example.com:8443 -u admin -p 'RedHat123!'
# 기대: Login Succeeded!

# oc-mirror 도구 확인
oc-mirror version

# pull-secret에 mirror.ocp1.example.com:8443 인증 포함 확인
jq '.auths | keys' ~/.docker/config.json
# "mirror.ocp1.example.com:8443"이 목록에 있어야 함
```

### 3.4.2 산출물 반입 및 무결성 검증

```bash
# 외장 디스크에서 Bastion으로 복사
mkdir -p /opt/oc-mirror-input
mount /dev/sdX1 /mnt/external

# 해시 검증
cd /mnt/external
sha256sum -c oc-mirror-output.sha256
# 기대: oc-mirror-output.tar: OK

# Bastion 작업 디렉토리로 복사
rsync -av --progress /mnt/external/oc-mirror-output/ /opt/oc-mirror-input/

# 또는 단일 tar를 받았다면 해제
tar -xf /mnt/external/oc-mirror-output.tar -C /opt/

ls /opt/oc-mirror-input/
# mirror_000001.tar  mirror_000002.tar  mirror_000003.tar  working-dir/
```

### 3.4.3 imageset-config.yaml 배치

**1단계에서 사용한 동일 imageset-config.yaml을 폐쇄망 Bastion에도 배치합니다.**

```bash
# 외부망에서 옮긴 imageset-config.yaml을 같은 위치에 둠
mkdir -p ~/oc-mirror-work
cp /opt/oc-mirror-input/imageset-config.yaml ~/oc-mirror-work/

# 또는 외부망 동일 내용으로 다시 작성
cat > ~/oc-mirror-work/imageset-config.yaml <<'EOF'
# ... (3.3.3과 동일 내용)
EOF
```

> **왜 같은 imageset-config.yaml이 필요한가**
> oc-mirror v2의 2단계 업로드는 imageset-config.yaml을 기준으로 tar 안의 어떤 이미지를 Mirror Registry의 어느 경로로 push할지 결정합니다. 1단계와 다른 설정을 사용하면 산출물이 일치하지 않거나 오류가 발생합니다.

### 3.4.4 업로드 실행

```bash
cd ~/oc-mirror-work

# Mirror Registry로 이미지 업로드
oc-mirror --config=imageset-config.yaml \
  --from=/opt/oc-mirror-input \
  docker://mirror.ocp1.example.com:8443 \
  --v2

# 옵션 설명:
# --config:    동일 imageset-config.yaml
# --from:      tar 아카이브가 있는 디렉토리
# docker://...: 업로드 대상 레지스트리 URL
# --v2:        v2 모드
```

업로드가 진행되며 다음과 같은 로그가 출력됩니다.

```
2026/05/15 09:00:00 [INFO]   : 🔂 Push progress: 1/850 (0.12%)
2026/05/15 09:00:30 [INFO]   : 🔂 Push progress: 50/850 (5.88%)
...
2026/05/15 11:00:00 [INFO]   : ✅ Push completed successfully
2026/05/15 11:00:01 [INFO]   : 📂 Cluster resources written to working-dir/cluster-resources/
```

### 3.4.5 업로드 결과 확인

```bash
# Mirror Registry에 push된 repository 목록 확인
curl -s -k -u admin:RedHat123! \
  https://mirror.ocp1.example.com:8443/v2/_catalog | jq

# 기대 출력 예시:
# {
#   "repositories": [
#     "openshift/release",
#     "openshift/release-images",
#     "kubernetes-nmstate-operator/...",
#     "kubevirt-hyperconverged/...",
#     "ubi9/ubi",
#     ...
#   ]
# }

# 특정 이미지의 tag 확인
curl -s -k -u admin:RedHat123! \
  https://mirror.ocp1.example.com:8443/v2/openshift/release/tags/list | jq
```

### 3.4.6 cluster-resources 산출물 확인

oc-mirror v2의 핵심 산출물인 `cluster-resources/` 디렉토리를 확인합니다.

```bash
ls ~/oc-mirror-work/working-dir/cluster-resources/
```

기대 출력:
```
idms-oc-mirror.yaml                                    # ImageDigestMirrorSet
itms-oc-mirror.yaml                                    # ImageTagMirrorSet
cs-redhat-operator-index-v4-20.yaml                    # CatalogSource for Red Hat Operators
signature-config-map-openshift-release-4.20.0.yaml     # Release image signature
release-signatures/                                    # 추가 signature
```

**ImageDigestMirrorSet (IDMS) 예시:**

```bash
cat working-dir/cluster-resources/idms-oc-mirror.yaml
```

```yaml
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: idms-release-0
spec:
  imageDigestMirrors:
  - mirrors:
    - mirror.ocp1.example.com:8443/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - mirror.ocp1.example.com:8443/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

이 리소스는 클러스터에게 "`quay.io/openshift-release-dev/ocp-release`를 요청하면 실제로는 `mirror.ocp1.example.com:8443/openshift/release-images`에서 pull해라"라고 알려줍니다.

**CatalogSource 예시:**

```bash
cat working-dir/cluster-resources/cs-redhat-operator-index-v4-20.yaml
```

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: cs-redhat-operator-index-v4-20
  namespace: openshift-marketplace
spec:
  displayName: Red Hat Operators (Mirror)
  image: mirror.ocp1.example.com:8443/redhat/redhat-operator-index:v4.20
  publisher: Red Hat
  sourceType: grpc
  updateStrategy:
    registryPoll:
      interval: 30m
```

이 리소스는 클러스터에게 Operator를 찾을 카탈로그 위치를 알려줍니다.

> **언제 이 매니페스트를 적용하는가**
> - **IDMS/ITMS**: 설치 시점 (`install-config.yaml`의 `imageDigestSources`로도 동일 기능) + 설치 후 추가 적용
> - **CatalogSource**: 설치 완료 후 (`oc apply -f`)
>
> 설치 시점에는 install-config.yaml의 imageDigestSources만 있어도 release image를 pull할 수 있습니다. 설치 후 IDMS/ITMS와 CatalogSource를 적용하면 Operator 설치가 가능해집니다.

---

## 3.5 install-config.yaml 작성

### 3.5.1 install-config.yaml의 역할

`install-config.yaml`은 OpenShift 설치의 핵심 입력 파일입니다. 설치 도구(`openshift-install`)가 이 파일을 읽어 클러스터의 모든 설정을 결정합니다.

생성된 후 다음과 같이 사용됩니다.
1. `install-config.yaml` 작성 → 검토
2. `openshift-install create manifests` → manifests 생성 (install-config.yaml 소비됨)
3. `openshift-install create ignition-configs` → ignition 생성

> **주의: install-config.yaml은 소비되면 사라진다**
> `openshift-install create manifests` 명령을 실행하면 install-config.yaml이 자동 삭제됩니다. 향후 변경/재실행 가능성에 대비해 **반드시 사본을 보관**하십시오.

### 3.5.2 전체 예시 파일

본 PoC의 전체 install-config.yaml:

```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: ocp1

# ============================================
# 컨트롤 플레인 (master)
# ============================================
controlPlane:
  name: master
  hyperthreading: Enabled
  replicas: 3
  architecture: amd64

# ============================================
# 워커 (compute) - UPI에서는 0으로 설정
# ============================================
compute:
- name: worker
  hyperthreading: Enabled
  replicas: 0      # UPI 방식이므로 0
  architecture: amd64

# ============================================
# 네트워킹
# ============================================
networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: 192.168.10.0/24          # Infra + NAS 네트워크만
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16

# ============================================
# 플랫폼 (UPI: none)
# ============================================
platform:
  none: {}

# ============================================
# Pull Secret
# ============================================
pullSecret: '<pull-secret-json-string>'

# ============================================
# SSH Key (RHCOS core 사용자에 등록됨)
# ============================================
sshKey: |
  ssh-ed25519 AAAA... user@bastion

# ============================================
# 미러 정보 - imageDigestSources
# ============================================
imageDigestSources:
- mirrors:
  - mirror.ocp1.example.com:8443/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - mirror.ocp1.example.com:8443/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev

# ============================================
# Mirror Registry CA 인증서
# ============================================
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIDxTCCAq2gAwIBAgIJAJxYMG2... (Mirror Registry CA 인증서)
  ...
  -----END CERTIFICATE-----

# ============================================
# 폐쇄망 명시 (선택)
# ============================================
publish: External   # 폐쇄망에서도 External 유지 (Internal은 클라우드 옵션)
```

### 3.5.3 핵심 필드 해설

**`baseDomain`과 `metadata.name`**

```yaml
baseDomain: example.com
metadata:
  name: ocp1
```

이 두 값이 결합되어 클러스터의 모든 도메인이 형성됩니다.
- API: `api.ocp1.example.com`
- Wildcard: `*.apps.ocp1.example.com`
- 노드: `master-0.ocp1.example.com`

DNS에 등록된 이름과 정확히 일치해야 합니다.

**`networking`**

```yaml
networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: 192.168.10.0/24
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
```

| 필드 | 의미 | 본 PoC 값 |
|---|---|---|
| `networkType` | CNI 플러그인 | OVNKubernetes (4.20 기본/유일) |
| `machineNetwork` | **노드 IP가 속한 네트워크 (Part 1.4 참조)** | 192.168.10.0/24만 |
| `clusterNetwork.cidr` | Pod IP 할당 범위 | 10.128.0.0/14 |
| `clusterNetwork.hostPrefix` | 노드당 Pod IP 서브넷 크기 | /23 (510개 Pod/노드) |
| `serviceNetwork` | Kubernetes Service ClusterIP 범위 | 172.30.0.0/16 |

> **다시 강조: machineNetwork에 Service망(192.168.20.0/24)을 절대 추가하지 말 것**
> Part 1.4에서 다룬 내용입니다. machineNetwork는 노드 InternalIP, OVN geneve 종단점, CSR 자동 승인 기준 등을 모두 결정합니다. Service망을 추가하면 OpenShift가 Service망 IP를 노드 통신에 사용할 수 있게 되어 광범위한 장애의 원인이 됩니다.

**`platform: none`**

```yaml
platform:
  none: {}
```

UPI 방식의 시그니처입니다. `aws`, `vsphere`, `baremetal` 등을 지정하면 IPI(자동 프로비저닝)로 동작하며, 노드 자동 생성을 시도합니다. `none`은 "사용자가 모든 인프라를 직접 준비한다"는 선언입니다.

**`pullSecret`**

3.2.4에서 준비한 통합 pull-secret을 한 줄 JSON 문자열로 넣습니다.

```bash
# pull-secret.json을 한 줄로 변환
jq -c . < ~/.docker/config.json
```

결과를 `pullSecret: '...'` 형식으로 install-config.yaml에 붙여넣습니다.

**`sshKey`**

설치된 노드에 SSH로 접근할 수 있도록 RHCOS의 `core` 사용자에 등록될 공개키입니다. 트러블슈팅에 매우 중요합니다.

```bash
# SSH 키 생성 (없다면)
ssh-keygen -t ed25519 -f ~/.ssh/ocp_id_ed25519 -N ""

# 공개키 내용을 sshKey에 붙여넣기
cat ~/.ssh/ocp_id_ed25519.pub
```

**`imageDigestSources`**

```yaml
imageDigestSources:
- mirrors:
  - mirror.ocp1.example.com:8443/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - mirror.ocp1.example.com:8443/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

이 항목이 폐쇄망 설치의 핵심입니다. 클러스터가 release image를 Mirror Registry에서 pull하도록 지시합니다. 3.4.6의 `idms-oc-mirror.yaml` 내용을 install-config.yaml 형식으로 옮긴 것과 같습니다.

oc-mirror v2의 cluster-resources에서 자동 추출:

```bash
# IDMS 내용을 install-config 형식으로 변환
cat ~/oc-mirror-work/working-dir/cluster-resources/idms-oc-mirror.yaml
# 위 내용의 imageDigestMirrors 부분을 install-config.yaml의 imageDigestSources로 복사
```

**`additionalTrustBundle`**

```yaml
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----
```

OpenShift 노드들이 Mirror Registry의 자체 서명 인증서를 신뢰하도록 CA 인증서를 포함시킵니다.

```bash
# Mirror Registry CA 인증서 내용을 그대로 붙여넣기
cat /opt/mirror-registry/quay/quay-rootCA/rootCA.crt
```

각 줄 앞에 2칸 들여쓰기 필수.

### 3.5.4 install-config.yaml 작성 자동화 스크립트

수동 편집이 번거로우므로 작성 헬퍼 스크립트를 만들어 둡니다.

```bash
cat > ~/build-install-config.sh <<'SCRIPT'
#!/bin/bash
set -e

WORK_DIR=~/openshift-install-work
mkdir -p $WORK_DIR

PULL_SECRET=$(jq -c . < ~/.docker/config.json)
SSH_KEY=$(cat ~/.ssh/ocp_id_ed25519.pub)
MIRROR_CA=$(cat /opt/mirror-registry/quay/quay-rootCA/rootCA.crt | sed 's/^/  /')

# IDMS에서 mirror 정보 추출
IDMS_MIRRORS=$(yq '.spec.imageDigestMirrors' \
  ~/oc-mirror-work/working-dir/cluster-resources/idms-oc-mirror.yaml | \
  sed 's/^/  /')

cat > $WORK_DIR/install-config.yaml <<EOF
apiVersion: v1
baseDomain: example.com
metadata:
  name: ocp1
controlPlane:
  name: master
  hyperthreading: Enabled
  replicas: 3
  architecture: amd64
compute:
- name: worker
  hyperthreading: Enabled
  replicas: 0
  architecture: amd64
networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: 192.168.10.0/24
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '$PULL_SECRET'
sshKey: |
  $SSH_KEY
imageDigestSources:
$IDMS_MIRRORS
additionalTrustBundle: |
$MIRROR_CA
publish: External
EOF

echo "Generated: $WORK_DIR/install-config.yaml"

# 사본 보관
cp $WORK_DIR/install-config.yaml $WORK_DIR/install-config.yaml.backup
echo "Backup: $WORK_DIR/install-config.yaml.backup"
SCRIPT

chmod +x ~/build-install-config.sh
~/build-install-config.sh
```

---

## 3.6 매니페스트 및 ignition 생성

### 3.6.1 매니페스트 생성

```bash
cd ~/openshift-install-work

# install-config.yaml의 사본 보관 (다음 명령에서 원본이 소비됨)
cp install-config.yaml install-config.yaml.backup

# 매니페스트 생성
openshift-install create manifests --dir=.

# 생성 결과 확인
ls -la
```

기대 출력:
```
manifests/
openshift/
install-config.yaml.backup
.openshift_install.log
.openshift_install_state.json
```

`install-config.yaml`은 사라지고 `manifests/`와 `openshift/` 디렉토리가 생성됩니다.

### 3.6.2 매니페스트 커스터마이징 (선택)

이 시점에 manifests를 수정하면 클러스터 초기 상태를 변경할 수 있습니다. 본 PoC에서는 다음을 고려할 수 있습니다.

**1. Master에 워크로드 스케줄링 금지 (이미 기본 동작이지만 명시화)**

```yaml
# manifests/cluster-scheduler-02-config.yml
apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  name: cluster
spec:
  mastersSchedulable: false   # 기본값, 명시화
```

**2. IDMS를 매니페스트에 미리 포함 (선택)**

oc-mirror가 만든 IDMS를 manifests에 미리 두면 설치 직후부터 적용됩니다.

```bash
cp ~/oc-mirror-work/working-dir/cluster-resources/idms-oc-mirror.yaml manifests/
cp ~/oc-mirror-work/working-dir/cluster-resources/itms-oc-mirror.yaml manifests/
```

단, install-config.yaml의 `imageDigestSources`가 이미 같은 역할을 하므로 중복 적용해도 무방합니다.

### 3.6.3 ignition 생성

```bash
cd ~/openshift-install-work

openshift-install create ignition-configs --dir=.
```

기대 출력:
```
ls -la
```
```
auth/
  ├── kubeadmin-password
  └── kubeconfig
bootstrap.ign
master.ign
worker.ign
metadata.json
```

| 파일 | 용도 |
|---|---|
| `bootstrap.ign` | Bootstrap 노드용 ignition |
| `master.ign` | 모든 Master 노드용 (동일) |
| `worker.ign` | 모든 Worker 노드용 (ap/db/podpool1 모두 동일) |
| `auth/kubeconfig` | 설치 직후 admin 접근용 kubeconfig |
| `auth/kubeadmin-password` | kubeadmin 초기 비밀번호 |
| `metadata.json` | 클러스터 메타데이터 |

> **kubeconfig와 kubeadmin-password는 매우 민감**
> `auth/` 디렉토리의 두 파일은 클러스터 전체 admin 권한입니다. 절대 외부에 노출되지 않도록 보관하고, 설치 완료 후 정식 인증 체계(LDAP, OAuth)를 구성한 다음 kubeadmin은 비활성화하는 것이 정석입니다.

### 3.6.4 ignition 파일을 Bastion 웹 서버로 배치

```bash
# httpd ignition 디렉토리에 복사
cp bootstrap.ign /var/www/html/ignition/
cp master.ign /var/www/html/ignition/
cp worker.ign /var/www/html/ignition/

# SELinux context 적용
chcon -t httpd_sys_content_t /var/www/html/ignition/*.ign

# 권한 (httpd가 읽을 수 있어야 함)
chmod 644 /var/www/html/ignition/*.ign

# 다른 호스트에서 접근 확인
curl http://192.168.10.10:8080/ignition/bootstrap.ign | jq . | head -20
```

기대: JSON 형식의 ignition 내용이 출력됨.

---

## 3.7 RHCOS 노드 설치

### 3.7.1 RHCOS Live ISO 준비

각 노드는 RHCOS Live ISO로 부팅한 후, `coreos-installer` 명령으로 OS를 디스크에 설치합니다.

**1. RHCOS 이미지 다운로드 (외부망 또는 미리 받아둠)**

```bash
# OpenShift 4.20 호환 RHCOS 이미지
# https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.20/
# 다음 파일 다운로드:
# - rhcos-4.20.x-x86_64-live.x86_64.iso
# - rhcos-4.20.x-x86_64-live-kernel-x86_64
# - rhcos-4.20.x-x86_64-live-initramfs.x86_64.img
# - rhcos-4.20.x-x86_64-live-rootfs.x86_64.img

# Bastion 웹 서버에 배치
mkdir -p /var/www/html/rhcos
cp rhcos-4.20.*-live* /var/www/html/rhcos/
chcon -R -t httpd_sys_content_t /var/www/html/rhcos
```

**2. 가상화 환경(vSphere, libvirt 등)에서 VM 생성**

각 노드 VM 사양:

| 노드 | CPU | Memory | Disk | NIC |
|---|---|---|---|---|
| bootstrap | 4 vCPU | 16 GB | 120 GB | 1 (Infra) |
| master-0/1/2 | 8 vCPU | 16 GB | 120 GB | 1 (Infra) |
| ap-worker-0/1 | 8 vCPU | 32 GB | 200 GB | 2 (Infra + Service) |
| db-worker-0/1 | 8 vCPU | 32 GB | 200 GB | 2 (Infra + Service) |
| podpool1-worker-0/1 | 8 vCPU | 32 GB | 200 GB | 2 (Infra + Service) |

각 VM은 RHCOS Live ISO를 마운트하여 부팅하도록 설정합니다.

### 3.7.2 Live ISO 부팅 후 네트워크 구성

Live ISO로 부팅하면 임시 콘솔이 뜹니다. 여기서 nmcli로 bond 구성을 수행합니다.

**Master 노드 (단일 NIC, bond 없음 - 단순 구성)**

Master는 NIC가 1개이므로 bond를 만들지 않고 단일 NIC로 직접 구성합니다.

```bash
# NIC 이름 확인
nmcli dev status

# Master-0 예시 (NIC: ens192)
nmcli con add type ethernet \
  con-name infra \
  ifname ens192 \
  ipv4.method manual \
  ipv4.addresses 192.168.10.31/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns "192.168.10.10" \
  ipv4.dns-search "ocp1.example.com" \
  connection.autoconnect yes

nmcli con up infra
```

> **Bond 구성을 권장하는 경우**
> 본 PoC는 단일 NIC 환경을 가정하므로 master에 bond를 만들지 않습니다. 실제 운영 환경에서 NIC 이중화가 필요하다면 nmcli로 bond0(active-backup 또는 LACP)를 만들고 그 위에 IP를 설정합니다.
>
> Bond 예시 (참고):
> ```
> nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup,miimon=100"
> nmcli con add type bond-slave con-name bond0-port1 ifname ens192 master bond0
> nmcli con add type bond-slave con-name bond0-port2 ifname ens224 master bond0
> nmcli con mod bond0 ipv4.method manual ipv4.addresses 192.168.10.31/24 ipv4.gateway 192.168.10.1 ipv4.dns 192.168.10.10
> ```

**AP/DB Worker 노드 (2 NIC: Infra + VM Service bridge)**

```bash
# NIC 확인 (예: ens192=Infra, ens224=Service)
nmcli dev status

# Infra NIC 구성
nmcli con add type ethernet \
  con-name infra \
  ifname ens192 \
  ipv4.method manual \
  ipv4.addresses 192.168.10.41/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns "192.168.10.10" \
  ipv4.dns-search "ocp1.example.com" \
  connection.autoconnect yes

# Service NIC 구성 - IP 없음, 추후 NMState로 bridge 생성
nmcli con add type ethernet \
  con-name service \
  ifname ens224 \
  ipv4.method disabled \
  ipv6.method disabled \
  connection.autoconnect yes

nmcli con up infra
nmcli con up service
```

> **AP/DB의 Service NIC에 IP를 설정하지 않는 이유**
> Part 1.6.2에서 다룬 대로, AP/DB 노드의 bond1(Service)은 호스트가 사용하는 망이 아니라 VM에게 제공하는 망입니다. 호스트 IP를 두지 않고, 추후 NMState NodeNetworkConfigurationPolicy(NNCP)로 Linux bridge를 생성합니다.

**PodPool1 Worker 노드 (2 NIC 모두 IP 보유)**

```bash
nmcli dev status

# Infra NIC (primary, default gateway 보유)
nmcli con add type ethernet \
  con-name infra \
  ifname ens192 \
  ipv4.method manual \
  ipv4.addresses 192.168.10.61/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns "192.168.10.10" \
  ipv4.dns-search "ocp1.example.com" \
  connection.autoconnect yes

# Service NIC - IP 있음, default gateway 없음
nmcli con add type ethernet \
  con-name service \
  ifname ens224 \
  ipv4.method manual \
  ipv4.addresses 192.168.20.61/24 \
  connection.autoconnect yes

# Service NIC의 자동 라우팅 비활성화 (default gateway 영향 방지)
nmcli con mod service ipv4.never-default yes

nmcli con up infra
nmcli con up service
```

**`ipv4.never-default yes`의 중요성**

이 옵션이 없으면 NetworkManager가 Service NIC의 DHCP/manual 설정에서 자동으로 default route를 추가하려 시도할 수 있습니다. `never-default`는 "이 connection은 default route를 만들지 않는다"를 명시합니다. PodPool1의 default gateway는 반드시 Infra NIC만으로 결정되어야 합니다.

### 3.7.3 네트워크 검증 (RHCOS 설치 전 필수)

설치 명령을 실행하기 전에 반드시 네트워크가 의도대로 동작하는지 검증합니다.

```bash
# IP 할당 확인
ip addr

# 라우팅 확인 (default가 Infra만 가리켜야 함)
ip route show default
# 기대: default via 192.168.10.1 dev ens192 ...

# DNS 조회
nslookup api.ocp1.example.com 192.168.10.10
# 기대: 192.168.10.20

nslookup api-int.ocp1.example.com 192.168.10.10
# 기대: 192.168.10.20

# Bastion 도달 확인
ping -c 3 192.168.10.10

# API LB 도달 확인 (TCP 6443)
nc -zv 192.168.10.20 6443
# 기대: Connection succeeded (HAProxy listen 중)

# MCS LB 도달 확인 (TCP 22623)
nc -zv 192.168.10.20 22623

# Ignition 서버 도달 확인
curl -s http://192.168.10.10:8080/ignition/ | head
```

> **이 검증을 건너뛰지 말 것**
> 네트워크 문제는 RHCOS 설치 후 부팅 단계에서 나타나며, 그 시점에는 디버깅이 매우 어렵습니다. Live ISO 상태에서 모든 통신이 의도대로 동작하는지 확인하는 것이 가장 빠른 길입니다.

### 3.7.4 coreos-installer 실행

네트워크 검증이 완료되면 디스크에 RHCOS를 설치합니다.

```bash
# 디스크 확인
lsblk
# 기대: /dev/sda 또는 /dev/vda 등

# 노드 종류별 ignition 파일 결정
# bootstrap: bootstrap.ign
# master-0/1/2: master.ign
# ap-worker-0/1, db-worker-0/1, podpool1-worker-0/1: worker.ign

# 설치 (master-0 예시)
sudo coreos-installer install /dev/sda \
  --ignition-url=http://192.168.10.10:8080/ignition/master.ign \
  --insecure-ignition \
  --copy-network

# 설치 옵션 설명:
# --ignition-url:      ignition 파일 위치 (Bastion HTTP)
# --insecure-ignition: HTTPS가 아닌 HTTP 사용 허용 (PoC는 HTTP)
# --copy-network:      Live ISO에서 구성한 NetworkManager profile을 설치된 OS로 복사 (핵심)

# 설치 완료 후 재부팅
sudo reboot
```

> **`--copy-network`의 중요성**
> 이 옵션을 빠뜨리면 Live ISO에서 구성한 네트워크가 사라지고, 재부팅 후 RHCOS는 DHCP로 IP를 받으려 시도합니다. 폐쇄망에 DHCP가 없으면 노드가 네트워크 없이 부팅되어 ignition도 받지 못하고, MCS와 통신할 수도 없습니다.
>
> **`--copy-network`는 UPI 정적 IP 환경에서 반드시 필요한 옵션**입니다.

### 3.7.5 ISO 분리 및 재부팅

```bash
# 설치 완료 후
# 1. VM 콘솔에서 Live ISO를 detach (vSphere/virt-manager에서)
# 2. VM을 재부팅
sudo reboot
```

재부팅 후 RHCOS가 디스크에서 부팅되고, 자동으로 ignition을 적용한 후 OpenShift 노드로 동작 시작합니다.

### 3.7.6 부팅 후 검증

각 노드가 부팅되면 SSH로 접근하여 상태를 확인할 수 있습니다.

```bash
# Bastion에서 SSH (sshKey가 설치되어 있어야 함)
ssh -i ~/.ssh/ocp_id_ed25519 core@192.168.10.31

# 또는 SSH config에 등록
cat >> ~/.ssh/config <<EOF
Host master-0
  HostName 192.168.10.31
  User core
  IdentityFile ~/.ssh/ocp_id_ed25519
  StrictHostKeyChecking no
EOF

ssh master-0

# 노드에서 확인
hostname
# 기대: master-0.ocp1.example.com

ip addr show ens192
ip route

# kubelet 상태
systemctl status kubelet

# 기대: bootstrap 완료 전까지는 Activating(start-pre) 상태일 수 있음
```

---

## 3.8 Bootstrap 및 설치 진행

### 3.8.1 노드 부팅 순서

UPI 설치의 표준 순서는 다음과 같습니다.

```
1. bootstrap 부팅
   ↓ (bootstrap이 임시 control plane 역할 시작)
2. master-0, master-1, master-2 부팅 (병렬 가능)
   ↓ (master 3대가 etcd 클러스터 형성, API 서버 ready)
3. bootstrap이 control plane 역할을 master에 위임 → bootstrap-complete
   ↓
4. worker 노드들 부팅 (ap, db, podpool1 모두)
   ↓ (각 worker가 CSR 발급 → 승인 → 노드 Ready)
5. install-complete
```

### 3.8.2 Bootstrap 부팅 모니터링

Bastion에서 설치 진행을 모니터링합니다.

```bash
cd ~/openshift-install-work

# bootstrap-complete 대기 (별도 터미널에서 실행)
openshift-install wait-for bootstrap-complete --dir=. --log-level=debug
```

기대 진행 로그:
```
INFO Waiting up to 20m0s for the Kubernetes API at https://api.ocp1.example.com:6443...
INFO API v1.31.x up
INFO Waiting up to 30m0s for bootstrapping to complete...
INFO Bootstrap status: complete
INFO Time elapsed: 25m12s
```

### 3.8.3 HAProxy 통계로 진행 상황 확인

```bash
# HAProxy 통계 페이지 (브라우저)
# http://192.168.10.10:9000

# 또는 CLI로
curl -s -u admin:RedHat123! 'http://192.168.10.10:9000/;csv' | \
  awk -F, '$2 != "FRONTEND" && $2 != "BACKEND" {print $1, $2, $18}' | \
  column -t
```

진행 단계별 backend 상태:
- bootstrap 부팅 직후: `bootstrap` UP, master들 DOWN
- master 부팅 후: `bootstrap` UP, `master-*` UP
- bootstrap-complete 후: bootstrap을 LB에서 제거

### 3.8.4 LB에서 bootstrap 제거

bootstrap-complete가 확인되면 즉시 LB에서 bootstrap을 제거합니다.

```bash
# Bastion에서
sudo sed -i 's/^    server bootstrap/#    server bootstrap/g' /etc/haproxy/haproxy.cfg

# HAProxy 설정 reload (무중단)
sudo systemctl reload haproxy

# 확인
ss -tlnp | grep haproxy
curl -s -u admin:RedHat123! 'http://192.168.10.10:9000/;csv' | grep bootstrap
# 기대: bootstrap 라인이 없거나 모두 disabled
```

bootstrap VM 자체는 다음과 같이 처리:
- PoC: VM 보존 (분석용)
- 프로덕션: VM 삭제 (자원 회수)

### 3.8.5 Worker CSR 자동 승인

Worker가 부팅되면 kubelet이 자신의 인증서를 발급해달라고 CSR(CertificateSigningRequest)을 만듭니다. `cluster-machine-approver`가 이를 자동 승인하지만, **UPI 환경에서는 특히 worker CSR에 대해 수동 승인이 필요한 경우가 있습니다.**

```bash
# kubeconfig 설정
export KUBECONFIG=~/openshift-install-work/auth/kubeconfig

# CSR 상태 확인
oc get csr

# Pending CSR이 있다면 일괄 승인
oc get csr -o name | xargs oc adm certificate approve

# 또는 한 줄로
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve

# CSR이 두 단계로 진행됨: kubelet-bootstrap → kubelet-serving
# 두 번 이상 승인이 필요할 수 있음 (worker가 Ready되기까지 보통 2~3회)
watch 'oc get csr; oc get nodes'
```

> **CSR이 자동 승인되지 않는 경우의 흔한 원인**
> 1. Worker의 InternalIP가 machineNetwork에 속하지 않음 (Part 1.4 위반)
> 2. 노드 hostname이 DNS에서 해석되지 않음 (Part 2.3 DNS 누락)
> 3. 시간 동기화 어긋남 (Part 2.6 chrony 미설정)
>
> 이 세 가지를 먼저 확인하십시오.

### 3.8.6 Install-complete 대기

```bash
cd ~/openshift-install-work

openshift-install wait-for install-complete --dir=. --log-level=debug
```

기대 결과:
```
INFO Waiting up to 40m0s for the cluster at https://api.ocp1.example.com:6443 to initialize...
INFO All cluster operators have completed progressing
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run
INFO     export KUBECONFIG=/root/openshift-install-work/auth/kubeconfig
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp1.example.com
INFO Login to the console with user: "kubeadmin", and password: "XXXX-XXXX-XXXX-XXXX"
```

설치 소요 시간 (실제):
- bootstrap → install-complete: 약 45~90분
- 네트워크/디스크 속도에 따라 변동

---

## 3.9 설치 완료 검증

### 3.9.1 클러스터 헬스 확인

```bash
export KUBECONFIG=~/openshift-install-work/auth/kubeconfig

# 노드 상태
oc get nodes -o wide
```

기대 출력:
```
NAME                STATUS   ROLES                  AGE   VERSION       INTERNAL-IP      EXTERNAL-IP
master-0            Ready    control-plane,master   1h    v1.31.x       192.168.10.31    <none>
master-1            Ready    control-plane,master   1h    v1.31.x       192.168.10.32    <none>
master-2            Ready    control-plane,master   1h    v1.31.x       192.168.10.33    <none>
ap-worker-0         Ready    worker                 30m   v1.31.x       192.168.10.41    <none>
ap-worker-1         Ready    worker                 30m   v1.31.x       192.168.10.42    <none>
db-worker-0         Ready    worker                 30m   v1.31.x       192.168.10.51    <none>
db-worker-1         Ready    worker                 30m   v1.31.x       192.168.10.52    <none>
podpool1-worker-0   Ready    worker                 30m   v1.31.x       192.168.10.61    <none>
podpool1-worker-1   Ready    worker                 30m   v1.31.x       192.168.10.62    <none>
```

**검증 포인트:**
- 모든 노드 STATUS=Ready
- 모든 worker INTERNAL-IP가 192.168.10.x (192.168.20.x가 아님 - Part 1.4 핵심)
- 모든 worker ROLES=worker (MCP 분리는 Part 4에서 수행)

### 3.9.2 ClusterOperator 상태

```bash
oc get clusterversion
```

기대:
```
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.20.0    True        False         15m     Cluster version is 4.20.0
```

```bash
oc get co
```

모든 ClusterOperator가 `AVAILABLE=True`, `PROGRESSING=False`, `DEGRADED=False` 이어야 합니다.

```bash
# 비정상 ClusterOperator만 추출
oc get co -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Available")].status}{"\t"}{.status.conditions[?(@.type=="Degraded")].status}{"\n"}{end}' | column -t
```

### 3.9.3 MachineConfigPool 상태

```bash
oc get mcp
```

기대:
```
NAME     CONFIG                        UPDATED   UPDATING   DEGRADED   ...
master   rendered-master-XXXXXXXX      True      False      False      ...
worker   rendered-worker-XXXXXXXX      True      False      False      ...
```

설치 직후에는 master와 worker MCP만 존재합니다. ap/db/podpool1 분리는 Part 4에서 수행합니다.

### 3.9.4 Pod 상태 확인

```bash
# 모든 namespace의 비정상 Pod 확인
oc get pods -A | grep -v -E 'Running|Completed'
```

비어있어야 정상입니다. 빌드 또는 Init 단계 Pod가 잠시 보일 수 있으나 곧 Running으로 전환됩니다.

### 3.9.5 Console 접속 확인

```bash
# kubeadmin 비밀번호 확인
cat ~/openshift-install-work/auth/kubeadmin-password

# 브라우저에서:
# https://console-openshift-console.apps.ocp1.example.com
# ID: kubeadmin
# PW: 위 파일 내용
```

> **Console 인증서 경고**
> 기본 설치 시 console 인증서는 자체 서명입니다. 브라우저에 경고가 표시되며, PoC에서는 그대로 진행해도 무방합니다. 정식 인증서로 교체하려면 Part 4에서 다룹니다.

---

## 3.10 IDMS/ITMS, CatalogSource 적용

설치 완료 후 oc-mirror가 생성한 cluster-resources를 적용하여 클러스터가 Operator 미러를 인식하도록 합니다.

### 3.10.1 cluster-resources 적용

```bash
cd ~/oc-mirror-work/working-dir/cluster-resources/

# 전체 리소스 적용
oc apply -f .
```

기대 출력:
```
imagedigestmirrorset.config.openshift.io/idms-release-0 created
imagedigestmirrorset.config.openshift.io/idms-operator-0 created
imagetagmirrorset.config.openshift.io/itms-generic-0 created
catalogsource.operators.coreos.com/cs-redhat-operator-index-v4-20 created
```

> **IDMS/ITMS 적용 시 MCP rolling update**
> IDMS/ITMS를 적용하면 master/worker 노드의 `/etc/containers/registries.conf`가 변경되므로 MCP rolling update가 발생합니다. 한 노드씩 재부팅되며 약 20~40분 소요됩니다. 설치 직후 한 번만 발생합니다.
>
> 설치 직후 임시 워크로드를 띄우지 말고 rolling update 완료 후 진행하는 것이 안전합니다.

### 3.10.2 적용 상태 확인

```bash
# IDMS 확인
oc get imagedigestmirrorset

# ITMS 확인
oc get imagetagmirrorset

# CatalogSource 확인
oc get catalogsource -n openshift-marketplace
```

CatalogSource는 ready되는 데 시간이 걸립니다.

```bash
# CatalogSource ready 대기
watch 'oc get catalogsource -n openshift-marketplace'
```

기대 (시간이 지난 후):
```
NAME                              DISPLAY                 TYPE   PUBLISHER   AGE
cs-redhat-operator-index-v4-20    Red Hat Operators (Mirror)   grpc   Red Hat     10m
```

### 3.10.3 MCP rolling update 모니터링

```bash
oc get mcp
```

UPDATING 상태가 보이면 진행 중입니다.

```bash
# 한 노드씩 재부팅 진행
watch 'oc get mcp; echo "---"; oc get nodes'
```

모든 MCP가 `UPDATED=True`, `UPDATING=False`가 되면 완료입니다.

### 3.10.4 미러 동작 검증

기본 카탈로그를 disable하고 미러 카탈로그만 사용하도록 설정합니다.

```bash
# 기본 Red Hat OperatorHub 카탈로그 비활성화 (폐쇄망에서 불필요)
oc patch operatorhub cluster --type=merge -p '{
  "spec": {
    "disableAllDefaultSources": true
  }
}'
```

PackageManifest 확인:

```bash
# 미러된 Operator 목록 조회
oc get packagemanifest -n openshift-marketplace
```

기대 출력에 다음이 포함되어야 합니다:
- kubernetes-nmstate-operator
- kubevirt-hyperconverged
- mtv-operator
- openshift-pipelines-operator-rh
- openshift-gitops-operator
- servicemeshoperator
- node-healthcheck-operator
- fence-agents-remediation

### 3.10.5 이미지 pull 테스트

```bash
# 테스트 Pod로 mirror 동작 확인
oc run mirror-test --image=registry.redhat.io/ubi9/ubi:latest -- sleep 3600

# Pod 이미지 origin 확인
oc describe pod mirror-test | grep -A1 Image:
# 기대: Image: registry.redhat.io/ubi9/ubi:latest
#       Image ID: mirror.ocp1.example.com:8443/ubi9/ubi@sha256:...

oc delete pod mirror-test
```

`Image ID`가 Mirror Registry 경로로 표시되면 미러가 정상 동작하는 것입니다.

---

## 3.11 트러블슈팅 가이드 (설치 단계)

### 3.11.1 흔한 문제와 해결

| 증상 | 가능한 원인 | 확인 명령 |
|---|---|---|
| `wait-for bootstrap-complete` 30분 이상 멈춤 | bootstrap이 master에 ignition 전달 실패 | bootstrap 콘솔에서 `journalctl -u bootkube` |
| master가 Ready 안 됨 | DNS 조회 실패, NTP 어긋남 | master에서 `nslookup api-int`, `chronyc tracking` |
| worker가 NotReady, CSR 자동 승인 안 됨 | InternalIP가 machineNetwork 밖 | `oc get nodes -o wide` |
| MCS 22623 timeout | LB backend에 master IP 잘못 등록 | HAProxy stats 확인 |
| 이미지 pull 실패 (`ImagePullBackOff`) | additionalTrustBundle 누락, mirror 인증 실패 | `oc describe pod` |
| Operator 설치 화면에 미러 카탈로그 안 보임 | CatalogSource 미적용, OperatorHub disable 안 됨 | `oc get catalogsource -A` |

### 3.11.2 bootstrap 트러블슈팅

bootstrap이 진행 중 멈췄다면 콘솔에서 직접 확인합니다.

```bash
# bootstrap 노드에 SSH
ssh -i ~/.ssh/ocp_id_ed25519 core@192.168.10.30

# bootkube 로그
sudo journalctl -u bootkube -f

# kubelet 로그
sudo journalctl -u kubelet -f

# release-image pull 시도
sudo crictl pull mirror.ocp1.example.com:8443/openshift/release-images:4.20.0-x86_64
```

### 3.11.3 worker 트러블슈팅

```bash
# Worker SSH
ssh -i ~/.ssh/ocp_id_ed25519 core@192.168.10.61

# 네트워크 확인
ip addr
ip route
nslookup api-int.ocp1.example.com

# kubelet 로그
sudo journalctl -u kubelet -f

# MCS 통신 확인
curl -k https://api-int.ocp1.example.com:22623/config/worker
# 기대: ignition 내용 출력
```

---

## 3.12 Part 3 학습 점검

다음 질문에 답할 수 있다면 Part 3를 충분히 학습한 것입니다.

1. oc-mirror v1과 v2의 가장 큰 차이는 무엇인가? 본 PoC가 v2를 사용하는 이유는?
2. 2단계 미러링에서 imageset-config.yaml이 1단계와 2단계 모두에서 동일하게 사용되는 이유는?
3. pull-secret에 Mirror Registry 인증 정보를 추가하는 이유는? Red Hat 인증만으로 안 되는가?
4. install-config.yaml의 `imageDigestSources`와 oc-mirror가 생성하는 `idms-oc-mirror.yaml`의 관계는?
5. `additionalTrustBundle`을 install-config.yaml에 포함시키는 이유는?
6. UPI 설치에서 `compute.replicas`를 0으로 설정하는 이유는?
7. `coreos-installer install` 명령에서 `--copy-network` 옵션을 빠뜨리면 어떻게 되는가?
8. AP/DB Worker의 Service NIC에 IP를 설정하지 않고, PodPool1 Worker의 Service NIC에는 IP를 설정하는 이유는?
9. PodPool1의 Service NIC에 `ipv4.never-default yes`를 설정하는 이유는?
10. bootstrap-complete 직후 LB에서 bootstrap을 제거하지 않으면 어떤 문제가 발생할 수 있는가?
11. Worker CSR이 자동 승인되지 않는 흔한 세 가지 원인은?
12. IDMS/ITMS를 적용하면 MCP rolling update가 발생하는 이유는?

---

*Part 3 끝. 다음은 Part 4 (Day-2 MCP 분리 및 MachineConfig) 입니다.*
