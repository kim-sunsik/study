# Part 3. 폐쇄망 미러링 및 UPI 설치

> **목적**  
> oc-mirror v2를 사용해 OpenShift 4.20 설치와 Day-2 Operator 설치에 필요한 이미지를 폐쇄망 Mirror Registry에 미러링하고, UPI 방식으로 클러스터를 설치합니다.

---

## 3.1 작업 흐름 요약

```text
1. 설치 도구 준비
2. Mirror Registry 로그인 및 CA 신뢰 구성
3. oc-mirror v2 ImageSetConfiguration 작성
4. OpenShift release / Operator / additional image 미러링
5. install-config.yaml 작성
6. manifests 및 ignition 생성
7. Ignition Web Server 배치
8. RHCOS Live ISO 부팅
9. nmcli 네트워크 구성
10. coreos-installer --copy-network 실행
11. bootstrap-complete 확인
12. bootstrap backend 제거
13. install-complete 확인
14. CSR 및 node InternalIP 검증
```

---

## 3.2 설치 도구 준비

Bastion에 다음 도구를 배치합니다.

```bash
mkdir -p ~/bin
export PATH=$HOME/bin:$PATH
```

필요 도구:

```text
- oc
- openshift-install
- oc-mirror
- coreos-installer, Live ISO에서 사용
```

검증:

```bash
oc version --client
openshift-install version
oc mirror version
```

---

## 3.3 Mirror Registry 준비

> 실제 Mirror Registry 설치 방식은 고객 환경에 따라 달라질 수 있습니다. 본 문서는 `mirror.ocp420.aonsoft.demo.com:8443`을 내부 레지스트리로 가정합니다.

### 3.3.1 로그인

```bash
podman login mirror.ocp420.aonsoft.demo.com:8443
```

검증:

```bash
curl -vk https://mirror.ocp420.aonsoft.demo.com:8443/v2/
```

정상 예:

```text
HTTP/1.1 200 OK
또는
HTTP/1.1 401 Unauthorized
```

`401 Unauthorized`는 인증이 필요하다는 의미이므로 Registry가 살아있는 상태로 볼 수 있습니다.

---

### 3.3.2 pull-secret 병합

Red Hat pull secret과 Mirror Registry 인증 정보를 병합합니다.

```bash
cp ~/pull-secret.json ~/pull-secret-merged.json
podman login mirror.ocp420.aonsoft.demo.com:8443 --authfile ~/pull-secret-merged.json
jq . ~/pull-secret-merged.json > /dev/null
```

---

### 3.3.3 CA 신뢰 구성

Mirror Registry CA를 Bastion에 신뢰시킵니다.

```bash
sudo mkdir -p /etc/pki/ca-trust/source/anchors
sudo cp mirror-registry-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust extract
```

검증:

```bash
curl https://mirror.ocp420.aonsoft.demo.com:8443/v2/
```

---

## 3.4 oc-mirror v2 ImageSetConfiguration

> 패키지명과 채널은 설치 시점의 Catalog를 반드시 확인해야 합니다. 아래는 PoC 기준 예시 골격입니다.

`imageset-config.yaml`

```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  platform:
    channels:
    - name: stable-4.20
      minVersion: 4.20.0
      maxVersion: 4.20.99
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:
    - name: kubevirt-hyperconverged
    - name: mtv-operator
    - name: openshift-pipelines-operator-rh
    - name: openshift-gitops-operator
    - name: servicemeshoperator
    - name: kubernetes-nmstate-operator
    - name: node-healthcheck-operator
    - name: fence-agents-remediation
  additionalImages:
  - name: registry.redhat.io/ubi9/ubi:latest
  - name: registry.redhat.io/jboss-eap-7/eap74-openjdk17-openshift-rhel8:latest
  - name: registry.redhat.io/jboss-eap-8/eap8-openjdk17-builder-openshift-rhel8:latest
```

패키지 확인 예:

```bash
oc mirror list operators \
  --catalog registry.redhat.io/redhat/redhat-operator-index:v4.20
```

---

## 3.5 미러링 실행

### 3.5.1 Mirror Registry로 직접 미러링

```bash
oc mirror \
  --config imageset-config.yaml \
  docker://mirror.ocp420.aonsoft.demo.com:8443
```

### 3.5.2 미러링 결과 확인

```bash
ls -al oc-mirror-workspace/
find oc-mirror-workspace -maxdepth 3 -type f | sort
```

설치 후 사용할 리소스는 일반적으로 workspace 내 `cluster-resources` 또는 유사 디렉토리에 생성됩니다.

---

## 3.6 install-config.yaml 작성

`~/ocp420-install/install-config.yaml`

```yaml
apiVersion: v1
baseDomain: aonsoft.demo.com
metadata:
  name: ocp420
platform:
  none: {}
networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: 192.168.10.0/24
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
pullSecret: '<pull-secret-json-string>'
sshKey: '<ssh-rsa ...>'
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  <MIRROR_REGISTRY_CA>
  -----END CERTIFICATE-----
imageDigestSources:
- mirrors:
  - mirror.ocp420.aonsoft.demo.com:8443/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - mirror.ocp420.aonsoft.demo.com:8443/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

> **주의**  
> oc-mirror v2 결과에 따라 `imageDigestSources` 또는 IDMS/ITMS 적용 방식이 달라질 수 있습니다. 설치 전 `oc mirror` 출력의 install-config 반영 값을 우선합니다.

---

## 3.7 manifests 및 ignition 생성

```bash
mkdir -p ~/ocp420-install
cp install-config.yaml ~/ocp420-install/
openshift-install create manifests --dir ~/ocp420-install
openshift-install create ignition-configs --dir ~/ocp420-install
```

생성 파일:

```text
bootstrap.ign
master.ign
worker.ign
metadata.json
```

Ignition 배치:

```bash
sudo mkdir -p /var/www/html/ocp420
sudo cp ~/ocp420-install/*.ign /var/www/html/ocp420/
sudo chmod 644 /var/www/html/ocp420/*.ign
curl http://192.168.10.10/ocp420/bootstrap.ign
```

---

## 3.8 RHCOS 노드 설치 전 네트워크 설정

> NIC 이름, bond mode, 디스크명은 환경에 맞게 반드시 확인합니다.

### 3.8.1 Master 예시

```bash
nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup,miimon=100"
nmcli con add type ethernet con-name bond0-port-eno1 ifname eno1 master bond0
nmcli con add type ethernet con-name bond0-port-eno2 ifname eno2 master bond0

nmcli con mod bond0 ipv4.method manual \
  ipv4.addresses 192.168.10.31/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns 192.168.10.10 \
  ipv4.dns-search ocp420.aonsoft.demo.com \
  connection.autoconnect yes

nmcli con up bond0
```

### 3.8.2 ap/db 예시

```bash
# bond0: Infra
nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup,miimon=100"
nmcli con add type ethernet con-name bond0-port-eno1 ifname eno1 master bond0
nmcli con add type ethernet con-name bond0-port-eno2 ifname eno2 master bond0

nmcli con mod bond0 ipv4.method manual \
  ipv4.addresses 192.168.10.41/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns 192.168.10.10 \
  ipv4.dns-search ocp420.aonsoft.demo.com \
  connection.autoconnect yes

# bond1: VM Service, no IP
nmcli con add type bond con-name bond1 ifname bond1 bond.options "mode=active-backup,miimon=100"
nmcli con add type ethernet con-name bond1-port-eno3 ifname eno3 master bond1
nmcli con add type ethernet con-name bond1-port-eno4 ifname eno4 master bond1
nmcli con mod bond1 ipv4.method disabled ipv6.method disabled connection.autoconnect yes

nmcli con up bond0
nmcli con up bond1
```

### 3.8.3 podpool1 예시

```bash
# bond0: Infra
nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup,miimon=100"
nmcli con add type ethernet con-name bond0-port-eno1 ifname eno1 master bond0
nmcli con add type ethernet con-name bond0-port-eno2 ifname eno2 master bond0

nmcli con mod bond0 ipv4.method manual \
  ipv4.addresses 192.168.10.61/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns 192.168.10.10 \
  ipv4.dns-search ocp420.aonsoft.demo.com \
  connection.autoconnect yes

# bond1: Service, no default gateway
nmcli con add type bond con-name bond1 ifname bond1 bond.options "mode=active-backup,miimon=100"
nmcli con add type ethernet con-name bond1-port-eno3 ifname eno3 master bond1
nmcli con add type ethernet con-name bond1-port-eno4 ifname eno4 master bond1

nmcli con mod bond1 ipv4.method manual \
  ipv4.addresses 192.168.20.61/24 \
  ipv4.never-default yes \
  connection.autoconnect yes

nmcli con up bond0
nmcli con up bond1
```

검증:

```bash
nmcli con show
ip addr
ip route
cat /etc/resolv.conf
ping -c 3 192.168.10.1
dig api.ocp420.aonsoft.demo.com
curl -k https://api.ocp420.aonsoft.demo.com:6443
```

---

## 3.9 coreos-installer 실행

### 3.9.1 Bootstrap

```bash
sudo coreos-installer install /dev/sda \
  --ignition-url http://192.168.10.10/ocp420/bootstrap.ign \
  --copy-network
sudo reboot
```

### 3.9.2 Master

```bash
sudo coreos-installer install /dev/sda \
  --ignition-url http://192.168.10.10/ocp420/master.ign \
  --copy-network
sudo reboot
```

### 3.9.3 Worker

```bash
sudo coreos-installer install /dev/sda \
  --ignition-url http://192.168.10.10/ocp420/worker.ign \
  --copy-network
sudo reboot
```

> **주의**  
> `/dev/sda`는 예시입니다. 실제 설치 대상 디스크를 `lsblk`로 반드시 확인해야 합니다.

---

## 3.10 Bootstrap 및 설치 완료 확인

### 3.10.1 Bootstrap 완료

```bash
openshift-install wait-for bootstrap-complete \
  --dir ~/ocp420-install \
  --log-level=debug
```

완료 후 Part 2 절차에 따라 HAProxy에서 bootstrap backend를 제거합니다.

### 3.10.2 설치 완료

```bash
openshift-install wait-for install-complete \
  --dir ~/ocp420-install \
  --log-level=debug
```

kubeconfig:

```bash
export KUBECONFIG=~/ocp420-install/auth/kubeconfig
```

검증:

```bash
oc get nodes -o wide
oc get clusterversion
oc get co
oc get mcp
```

---

## 3.11 CSR 승인

```bash
oc get csr
```

수동 승인:

```bash
oc adm certificate approve <csr-name>
```

PoC 일괄 승인:

```bash
oc get csr -o name | xargs -I{} oc adm certificate approve {}
```

> 운영에서는 CSR 요청자와 노드명을 확인한 뒤 승인하는 절차를 권장합니다.

---

## 3.12 node InternalIP 검증

```bash
oc get nodes -o wide
oc describe node podpool1-worker-0 | grep -A5 Addresses
oc describe node podpool1-worker-1 | grep -A5 Addresses
```

기대값:

```text
podpool1-worker-0 InternalIP = 192.168.10.61
podpool1-worker-1 InternalIP = 192.168.10.62
```

문제 상황:

```text
InternalIP = 192.168.20.61 또는 192.168.20.62
```

확인:

```bash
ip route
nmcli con show bond0
nmcli con show bond1
cat /etc/systemd/system/kubelet.service.d/20-nodenet.conf
```

---

## 3.13 oc-mirror 결과 리소스 적용

설치 완료 후 oc-mirror workspace의 cluster resources를 적용합니다.

```bash
oc apply -f oc-mirror-workspace/working-dir/cluster-resources/
```

검증:

```bash
oc get imagedigestmirrorset
oc get imagetagmirrorset
oc get catalogsource -A
oc get packagemanifest -n openshift-marketplace
```

---

## 3.14 실패 시 확인

| 증상 | 확인 |
|---|---|
| bootstrap-complete 지연 | HAProxy 6443/22623, DNS, bootstrap journal |
| master 부팅 실패 | ignition URL, MCS 접근, DNS |
| worker NotReady | CSR, node IP, MCS 접근 |
| image pull 실패 | pull-secret, mirror registry, CA |
| x509 오류 | additionalTrustBundle, mirror CA |
| InternalIP 오류 | default route, `ipv4.never-default`, machineNetwork |
| CatalogSource 없음 | oc-mirror 결과 리소스 적용 여부 |

---

## 3.15 Part 3 학습 점검

1. install-config에서 `machineNetwork`를 `192.168.10.0/24`만 지정하는 이유는 무엇인가?
2. `--copy-network` 옵션을 사용하는 이유는 무엇인가?
3. bootstrap 완료 후 HAProxy에서 제거해야 하는 backend는 무엇인가?
4. CSR 승인이 필요한 이유는 무엇인가?
5. oc-mirror v2 결과물에서 클러스터에 적용해야 하는 리소스는 무엇인가?

---

*Part 3 끝. 다음은 Part 4 — Day-2 MCP 및 MachineConfig입니다.*
