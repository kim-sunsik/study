# OpenShift 4.20 폐쇄망 UPI 설치 및 시나리오 검증 교육 가이드

> **본 가이드는 OpenShift Container Platform 4.20을 폐쇄망(에어갭) 환경에 UPI 방식으로 설치하고, VM과 컨테이너 워크로드를 분리 운영하며, MTV 마이그레이션과 FAR fencing 시나리오를 검증하는 PoC 가이드입니다.**
>
> 단순 절차 모음이 아니라 **"왜 그렇게 설계하는가"를 학습하는 교육자료**를 지향합니다.

---

## 본 가이드의 특징

### 교육자료 지향
- 각 결정에 대한 **배경과 대안 비교** 명시
- "왜 그렇게 하는가" 박스로 의사결정 근거 설명
- "흔한 실수" 박스로 학습자가 빠지기 쉬운 함정 안내
- 각 파트 끝에 학습 점검 질문 제공

### PoC와 프로덕션 분리 명시
- 본 가이드는 PoC 환경 기준으로 작성
- 프로덕션 권장 구성을 별도 박스로 명시
- PoC → 프로덕션 전환 시 변경할 항목 일괄 정리 (Part 6)

### 검증 가능한 절차
- 모든 명령에 기대 출력 명시
- 단계별 검증 명령 제공
- 트러블슈팅 가이드 포함

---

## 핵심 설계 원칙

본 가이드 전체를 관통하는 다섯 가지 원칙입니다.

1. **관리 트래픽은 Infra망, 서비스 트래픽은 Service망** — 트래픽 종류와 네트워크의 1:1 대응
2. **Master는 control plane 전용** — Service망에 연결하지 않음
3. **VM 노드와 컨테이너 노드는 분리** — ap/db는 VM 전용, podpool1은 컨테이너 전용
4. **PoC와 프로덕션 구성을 문서에서 명시적으로 분리** — 전환 시 변경점 자동 추적
5. **NAS는 PoC에서는 Infra와 통합, 프로덕션에서는 분리 검토** — 자원 제약과 분리 원칙의 균형

---

## 가이드 구성

본 가이드는 6개 파트로 구성되어 있으며, 총 8,200줄 분량의 교육자료입니다.

### Part 1. [아키텍처 및 네트워크 설계](./01-architecture-and-network.md) (696줄)

설계 결정을 다루는 가장 중요한 파트. 이후 모든 파트는 Part 1의 결정을 구체화합니다.

- 문서 범위와 학습 목표
- 전체 아키텍처와 노드 구성 (Bastion, Master 3, AP×2, DB×2, PodPool1×2)
- 네트워크 분리 원칙 (Infra+NAS / Service)
- 두 사이클 라우팅 (관리 사이클, 서비스 사이클)
- **machineNetwork 심화** — 노드 InternalIP, OVN 터널, CSR 자동 승인 결정
- IP 대역 설계 (192.168.10.0/24, 192.168.20.0/24)
- 노드별 NIC/Bond 설계 (Master, AP/DB, PodPool1)
- Default Gateway 정책 (모든 노드 Infra)
- Ingress 설계 (default = ap, service = podpool1, 도메인 분리)
- Egress 설계 (EgressIP 우선, 정책 라우팅 fallback)
- PoC vs 프로덕션 차이 요약

### Part 2. [Bastion 및 Load Balancer](./02-bastion-and-lb.md) (1,113줄)

Bastion 단일 노드에 모든 의존 서비스를 통합 구축합니다.

- Bastion 역할과 단일점 인식
- 네트워크 구성 (Infra + Service NIC, VIP 할당)
- BIND DNS 구성 (정방향/역방향 zone 파일 전체 예시)
- HAProxy 전체 설정 (6개 frontend: API, MCS, Default Ingress, Service Ingress)
- HAProxy 설정 요소 해설 (mode tcp, 1936/healthz 헬스체크 등)
- mirror-registry 설치
- NTP/Chrony 위계 설계 (Bastion 기준)
- Web Server (Ignition 제공)
- oc/openshift-install/oc-mirror 도구 배치
- vBMC 사전 준비
- 통합 검증 절차

### Part 3. [폐쇄망 미러링 및 OpenShift 설치](./03-mirror-and-install.md) (1,768줄)

oc-mirror v2 2단계 미러링과 UPI 설치 전체 흐름입니다.

- oc-mirror v2 개요 (v1과의 차이, IDMS/ITMS vs ICSP)
- ImageSet 설계 (imageset-config.yaml 전체 예시)
- pull-secret 준비 (Red Hat + Mirror Registry 인증 통합)
- **1단계** — 외부망에서 미러링 (`file://output`)
- **2단계** — 폐쇄망에서 Mirror Registry 업로드 (`docker://`)
- cluster-resources 산출물 (IDMS, ITMS, CatalogSource)
- install-config.yaml 전체 해설 (networking, platform: none, imageDigestSources, additionalTrustBundle)
- 매니페스트와 ignition 생성
- RHCOS Live ISO 부팅과 nmcli 네트워크 구성 (노드별 차이)
- `--copy-network`와 `ipv4.never-default` 옵션의 중요성
- Bootstrap → Master → Worker 순차 부팅
- Worker CSR 자동 승인
- 설치 완료 검증 (nodes, co, mcp)
- IDMS/ITMS/CatalogSource 적용과 MCP rolling update

### Part 4. [Day-2 MCP 분리 및 MachineConfig](./04-day2-mcp-and-mc.md) (1,314줄)

워크로드 격리를 실제로 구현하는 MCP 분리와 차등 MC 적용입니다.

- MCP/MC 개념과 MCO 동작 흐름
- "MC 변경 = rolling reboot" 비용 인식
- ap/db/podpool1 MCP YAML 정의 (machineConfigSelector, nodeSelector 해설)
- 노드 라벨링과 MCP 자동 편입
- worker 라벨 유지 vs 제거 trade-off
- MachineConfig 적용 — **chrony** (Bastion NTP)
- MachineConfig 적용 — **kdump** (crashkernel=512M)
- MachineConfig 적용 — **multipath** (ap/db만)
- MC 통합 적용 전략 (rolling reboot 1회로 줄이기)
- 인증서 관리 (API, Ingress 외부 인증서 교체)
- etcd 백업 정책 (NAS 보관)
- 노드 디스크 설계 (/var 분리)

### Part 5. [Operator 설치 및 OpenShift Virtualization](./05-operators-and-virtualization.md) (1,746줄)

플랫폼 구성 요소를 의존성 순서대로 설치합니다.

- OLM 6가지 핵심 리소스
- Operator 설치 권장 순서 (의존성 기반)
- **NMState Operator** + NodeNetworkConfigurationPolicy
- **VM Service Bridge 구성** (ap/db의 ens224 → br-vm-svc)
- bridge에 IP를 부여하지 않는 이유
- **OpenShift Virtualization** + HyperConverged CR
- `infra.nodePlacement`와 `workloads.nodePlacement` 분리
- virt-handler가 podpool1에 가지 않게 만드는 affinity
- **NetworkAttachmentDefinition (NAD)** + cnv-bridge
- 테스트 VM (Fedora cloud-init)
- **NAS NFS StorageClass** (수동 PV 방식)
- **IngressController 분리** (default = ap, service = podpool1)
- 두 wildcard 도메인 (`*.apps` vs `*.svcapps`)
- 추가 Operator (MTV, Pipelines, GitOps, Service Mesh, NHC, FAR)

### Part 6. [시나리오 테스트와 운영 통합](./06-mtv-far-workload-troubleshooting.md) (1,628줄)

본 PoC의 핵심 시나리오 검증과 운영 전환 준비입니다.

- **MTV 시나리오** (vSphere Provider, NetworkMap, StorageMap, Plan, Migration)
- **FAR + vBMC fencing 시나리오** (vBMC 등록, FAR Template, NHC, 강제 정지 테스트)
- **업무 워크로드 배치** (Tekton, GitOps, Service Mesh, JBoss EAP 8)
- **EgressIP 검증** (secondary NIC 할당, 동작 확인)
- **종합 검증 체크리스트**
- **트러블슈팅 가이드 (18가지)**
- **PoC → 프로덕션 전환 체크리스트**
- 학습 자료 및 다음 단계

---

## 추천 학습 순서

### 1차 학습: 개념 이해 (Part 1, Part 2 중심)
- Part 1 전체 읽기 (설계 결정의 배경 파악)
- Part 2의 DNS, HAProxy 부분 (관리 인프라 구조 이해)
- 각 파트 끝의 학습 점검 질문 답해보기

### 2차 학습: 설치 실습 (Part 2~3)
- Part 2 따라 Bastion 구축
- Part 3의 미러링 절차 수행
- install-config 작성 및 OpenShift 설치
- 설치 완료 검증

### 3차 학습: Day-2 운영 (Part 4~5)
- Part 4 따라 MCP 분리, MachineConfig 적용
- Part 5 따라 Operator 설치, Virtualization 구성
- IngressController 분리 검증

### 4차 학습: 시나리오 검증 (Part 6)
- MTV 마이그레이션 시나리오
- FAR/vBMC fencing 시나리오
- EgressIP 동작 검증
- 종합 체크리스트로 클러스터 전체 상태 점검

---

## 환경 요구사항 요약

### 노드 사양

| 노드 | 수량 | CPU | Memory | Disk | NIC |
|---|---|---|---|---|---|
| Bastion | 1 | 8 vCPU | 32 GB | 200 GB OS + 1 TB Mirror | 2 (Infra + Service) |
| Master | 3 | 8 vCPU | 16 GB | 120 GB | 1 (Infra) |
| AP Worker | 2 | 8 vCPU | 32 GB | 200 GB | 2 (Infra + Service) |
| DB Worker | 2 | 8 vCPU | 32 GB | 200 GB | 2 (Infra + Service) |
| PodPool1 Worker | 2 | 8 vCPU | 32 GB | 200 GB | 2 (Infra + Service) |
| **합계 (Bastion 제외)** | **9** | **64 vCPU** | **224 GB** | **1.5 TB** | |

### 네트워크

| 네트워크 | CIDR | 용도 |
|---|---|---|
| Infra + NAS | 192.168.10.0/24 | OpenShift 관리, etcd, NAS |
| Service | 192.168.20.0/24 | 업무 Ingress/Egress, VM 서비스 |
| Pod Network | 10.128.0.0/14 | Pod 내부 통신 (OVN) |
| Kubernetes Service CIDR | 172.30.0.0/16 | Service ClusterIP |

### 도메인

| 용도 | 도메인 |
|---|---|
| API | `api.ocp1.example.com` |
| API 내부 | `api-int.ocp1.example.com` |
| 관리 Ingress 와일드카드 | `*.apps.ocp1.example.com` |
| 업무 Ingress 와일드카드 | `*.svcapps.ocp1.example.com` |

### 외부망 워크스테이션 (미러링용)

- 인터넷 접속 가능
- 4 core / 16 GB / 1 TB
- `registry.redhat.io` 접근 가능
- Red Hat pull-secret 보유

---

## 사용 도구

| 도구 | 버전 | 용도 |
|---|---|---|
| openshift-install | 4.20.x | 클러스터 설치 |
| oc | 4.20.x | 클러스터 관리 CLI |
| oc-mirror | v2 | 폐쇄망 미러링 |
| virtctl | OCP 4.20 호환 | VM 관리 CLI |
| ipmitool | 1.8+ | vBMC fencing 테스트 |
| haproxy | 2.4+ | Load Balancer |
| bind | 9.x | DNS |
| chrony | 4.x | NTP |
| mirror-registry | (Red Hat 제공) | Quay 기반 Mirror Registry |

---

## 미러링 대상 Operator

본 PoC에서 검증할 Operator 8개:

1. Kubernetes NMState Operator
2. OpenShift Virtualization Operator
3. Migration Toolkit for Virtualization (MTV)
4. OpenShift Pipelines (Tekton)
5. OpenShift GitOps (ArgoCD)
6. Red Hat OpenShift Service Mesh
7. Node Health Check Operator
8. Fence Agents Remediation Operator

추가 이미지:
- UBI 9 (ubi, ubi-minimal)
- JBoss EAP 7, JBoss EAP 8

---

## 자주 참고하는 표

### MCP별 역할

| MCP | 노드 수 | 워크로드 | 외부 노출 |
|---|---|---|---|
| master | 3 | control plane (API, etcd) | API VIP |
| ap | 2 | AP성 VM + Default IngressController (PoC 임시) | apps 도메인 (PoC) |
| db | 2 | DB성 VM | - |
| podpool1 | 2 | 컨테이너 업무 + Service IngressController | svcapps 도메인 |

### LB VIP 매트릭스

| 용도 | VIP | 포트 | LB |
|---|---|---|---|
| API | 192.168.10.20 | 6443 | Bastion HAProxy |
| MCS | 192.168.10.20 | 22623 | Bastion HAProxy |
| Default Ingress (관리) | 192.168.10.21 | 80, 443 | Bastion HAProxy |
| Service Ingress (업무) | 192.168.20.20 | 80, 443 | Bastion HAProxy (별도 frontend) |

### Pod 배치 매트릭스

| 워크로드 | 배치 노드 | 메커니즘 |
|---|---|---|
| OpenShift control plane | master | OpenShift 기본 |
| OCP-V infra (virt-api 등) | master (PoC) / infra MCP (프로덕션) | HyperConverged `infra.nodePlacement` |
| OCP-V workload (virt-handler 등) | ap, db | HyperConverged `workloads.nodePlacement` affinity |
| VM (VirtualMachine) | ap, db | VM CR의 `nodeSelector` |
| Default IngressController router | ap (PoC) / infra (프로덕션) | IngressController `nodePlacement` |
| Service IngressController router | podpool1 | IngressController `nodePlacement` |
| 업무 컨테이너 Pod | podpool1 | namespace `openshift.io/node-selector` |

---

## 트러블슈팅 빠른 참조

| 증상 | 가능한 원인 | 참고 |
|---|---|---|
| Mirror 이미지 pull 실패 | pull-secret 또는 additionalTrustBundle | Part 6.6 #1, #17 |
| DNS 해석 실패 | BIND zone 파일 또는 노드 DNS 설정 | Part 6.6 #2 |
| API/MCS 도달 불가 | HAProxy 또는 SELinux | Part 6.6 #3 |
| bootstrap-complete 지연 | bootstrap 노드 디버깅 | Part 6.6 #4 |
| Worker NotReady | CSR 자동 승인 누락 | Part 6.6 #5 |
| 재부팅 후 IP 사라짐 | --copy-network 누락 | Part 6.6 #6 |
| MCP UPDATING 멈춤 | MachineConfig 오류 | Part 6.6 #7 |
| NNCP 적용 후 SSH 끊김 | NMState 자동 rollback 대기 | Part 6.6 #9 |
| VM 네트워크 안 됨 | bridge 또는 스위치 VLAN | Part 6.6 #10 |
| NAS PVC Pending | PV 부족 또는 accessMode | Part 6.6 #12 |
| MTV Plan Ready=False | NetworkMap/StorageMap 오류 | Part 6.1.10 |
| Fencing 동작 안 함 | vBMC 또는 IPMI 설정 | Part 6.2.9 |

---

## 라이선스 및 책임

본 가이드는 교육 목적으로 작성되었습니다. 본 가이드 내용 그대로 운영 환경에 적용하기 전에 다음을 반드시 검토하십시오.

1. 실제 OpenShift 4.20 공식 문서와의 일치 여부 (버전 z-stream에 따라 일부 옵션 변경 가능)
2. 사내 보안 정책과의 호환성
3. Red Hat 구독 및 라이선스 조건
4. 사용자 환경의 하드웨어/네트워크 특수성

본 가이드의 각 결정은 일반적인 PoC 환경을 가정한 것이며, 운영 환경에서는 사내 표준과 보안 요구사항에 맞춰 조정되어야 합니다.

---

## 가이드 완성도

| Part | 분량 | 상태 |
|---|---|---|
| Part 1 | 696줄 | 완료 |
| Part 2 | 1,113줄 | 완료 |
| Part 3 | 1,768줄 | 완료 |
| Part 4 | 1,314줄 | 완료 |
| Part 5 | 1,746줄 | 완료 |
| Part 6 | 1,628줄 | 완료 |
| **합계** | **8,265줄** | **6/6 완료** |

---

*본 가이드는 OpenShift 4.20 PoC 교육용 자료로 제작되었습니다.*
