# Kubespray 폐쇄망 설치 가이드

폐쇄망(Air-Gapped) 환경에서 Kubespray를 사용하여 Kubernetes 클러스터를 배포하고 운영하기 위한 가이드입니다.

## 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           폐쇄망 (Air-Gapped Network)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐       ┌──────────────────────────────────────────┐   │
│  │ Ansible 컨트롤러  │       │           Kubernetes 클러스터             │   │
│  │                  │  SSH  │  ┌────────────────────────────────────┐  │   │
│  │  - Kubespray     │──────►│  │      Control Plane (x3, HA)       │  │   │
│  │  - Python venv   │       │  │   kube-apiserver + etcd (Raft)    │  │   │
│  └──────────────────┘       │  └────────────────────────────────────┘  │   │
│           │                 │  ┌────────────────────────────────────┐  │   │
│           │                 │  │       Worker Nodes (x20)           │  │   │
│           │                 │  │   kubelet + containerd + Cilium    │  │   │
│           │                 │  └────────────────────────────────────┘  │   │
│           │                 └──────────────────────────────────────────┘   │
│           │                                    ▲                           │
│           ▼                                    │ Pull                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    레포지토리 서버 (Repo Server)                       │  │
│  │  ┌──────────────────────────┐  ┌─────────────────────────────────┐  │  │
│  │  │   프록시 레지스트리        │  │       파일 서버 (Nginx)          │  │  │
│  │  │  :5000 docker.io         │  │  :8080 바이너리 파일             │  │  │
│  │  │  :5001 quay.io           │  │   ├─ kubeadm, kubelet, kubectl  │  │  │
│  │  │  :5002 ghcr.io           │  │   ├─ etcd, containerd, crictl   │  │  │
│  │  │  :5003 registry.k8s.io   │  │   └─ helm, cni-plugins          │  │  │
│  │  └──────────────────────────┘  └─────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 기술 스택

| 구성 요소 | 기술 | 역할 |
|-----------|------|------|
| **Cluster Provisioning** | Kubespray (Ansible) | Kubernetes 클러스터 자동 배포 |
| **Container Runtime** | containerd | OCI 컨테이너 실행 |
| **CNI** | Cilium (eBPF) | 네트워킹, 로드밸런싱, 보안 정책 |
| **Distributed Storage** | etcd (Raft) | 클러스터 상태 저장 |
| **Control Plane HA** | Stacked etcd | 3노드 고가용성 구성 |
| **Registry Proxy** | Registry:2 (Podman) | 컨테이너 이미지 프록시 캐싱 |
| **File Server** | Nginx | Kubernetes 바이너리 호스팅 |

## Day 0 / Day 1 / Day 2 운영 모델

```
     Day 0                    Day 1                    Day 2
  ┌──────────┐           ┌──────────┐           ┌──────────┐
  │  설계 및  │  ──────►  │  설치 및  │  ──────►  │  운영 및  │
  │   준비   │           │   배포   │           │  유지보수  │
  └──────────┘           └──────────┘           └──────────┘
       │                      │                      │
       ▼                      ▼                      ▼
  • 아키텍처 설계         • 클러스터 설치         • 업그레이드
  • 소스 확보            • CNI 배포             • 노드 관리
  • 환경 구성            • 애드온 설치          • 트러블슈팅
  • 버전 매트릭스        • 검증                • 모니터링
```

## 프로젝트 구조

```
kubespray-offline-install-guide/
├── docs/
│   ├── day0-preparation/           # Day 0: 설계 및 준비
│   │   ├── 01-kubespray-version-guide.md
│   │   ├── 02-rhel10-install.md
│   │   ├── 03-kubespray-offline-prepare.md
│   │   ├── 04-cilium-offline-prepare.md
│   │   ├── 05-proxy-container-registry.md
│   │   └── 06-version-matrix.md
│   ├── day1-deployment/            # Day 1: 설치 및 배포
│   │   ├── 01-cluster-install.md
│   │   └── 02-cilium-install.md
│   ├── day2-operations/            # Day 2: 운영 및 유지보수
│   │   ├── 01-upgrade-guide.md
│   │   └── upgrade-notes/
│   │       └── 2.28-to-2.29.md
│   ├── concepts/                   # 공학적 개념 심화 학습
│   │   ├── 00-overview.md          # 학습 로드맵
│   │   ├── 01-linux-fundamentals.md
│   │   ├── 02-container-fundamentals.md
│   │   ├── 03-networking-fundamentals.md
│   │   ├── 04-kubernetes-architecture.md
│   │   ├── 05-distributed-systems.md
│   │   ├── 06-ebpf-cilium.md
│   │   └── 07-offline-infrastructure.md
│   └── engineering-concepts.md     # 공학적 개념 인덱스
├── configs/                        # 샘플 설정 파일
│   ├── cilium/
│   │   └── cilium-values-custom.yaml
│   └── kubespray/
│       ├── custom-config.yml
│       └── inventory.ini
├── playbooks/                      # Ansible Playbooks
│   ├── offline-pre-setup.yml
│   └── verify-offline-resources.yml
└── sources/                        # 소스 파일
    ├── kubespray/
    │   ├── kubespray-2.28.1.tar.gz
    │   └── kubespray-2.29.1.tar.gz
    └── cilium/
        ├── cli/                    # Cilium CLI 바이너리
        ├── helm-charts/            # Helm Charts (.tgz)
        └── values/                 # 버전별 Values 파일
```

## 시작하기

### 사전 요구 사항

| 항목 | 요구사항 | 비고 |
|------|---------|------|
| **OS** | RHEL 10 (커널 6.12+) | Cilium netkit 데이터패스 지원 |
| **Python** | 3.x | Kubespray 버전별 venv 권장 |
| **Ansible** | requirements.txt 참조 | Kubespray 버전별 상이 |
| **네트워크** | SSH 접근 가능 | 모든 노드 |

### 클러스터 구성 예시

| 역할 | 노드 수 | IP 범위 | 설명 |
|------|--------|---------|------|
| Control Plane | 3 | 192.168.108.11-13 | Stacked etcd (HA) |
| Worker | 20 | 192.168.108.21-40 | 워크로드 실행 |
| Repo Server | 1 | 192.168.108.201 | 레지스트리 + 파일 서버 |

### 네트워크 CIDR 설계

| 네트워크 | CIDR | 용도 |
|----------|------|------|
| 물리 네트워크 | 192.168.108.0/24 | 노드 통신 |
| Pod 네트워크 | 10.200.0.0/16 | Cilium IPAM (노드당 /24) |
| Service 네트워크 | 10.201.0.0/18 | ClusterIP 서비스 |

## 문서 가이드

### Day 0: 준비 단계

| # | 문서 | 설명 |
|---|------|------|
| 1 | [01-kubespray-version-guide.md](docs/day0-preparation/01-kubespray-version-guide.md) | Git 태그 기반 소스 확보 |
| 2 | [02-rhel10-install.md](docs/day0-preparation/02-rhel10-install.md) | RHEL 10 설치 및 Subscription |
| 3 | [03-kubespray-offline-prepare.md](docs/day0-preparation/03-kubespray-offline-prepare.md) | 오프라인 바이너리 준비 |
| 4 | [04-cilium-offline-prepare.md](docs/day0-preparation/04-cilium-offline-prepare.md) | Cilium CLI/Helm Chart 준비 |
| 5 | [05-proxy-container-registry.md](docs/day0-preparation/05-proxy-container-registry.md) | 프록시 레지스트리 구성 |
| 6 | [06-version-matrix.md](docs/day0-preparation/06-version-matrix.md) | 버전 호환성 매트릭스 |

### Day 1: 배포 단계

| # | 문서 | 설명 |
|---|------|------|
| 1 | [01-cluster-install.md](docs/day1-deployment/01-cluster-install.md) | Kubernetes 클러스터 설치 |
| 2 | [02-cilium-install.md](docs/day1-deployment/02-cilium-install.md) | Cilium CNI 설치 |

### Day 2: 운영 단계

| # | 문서 | 설명 |
|---|------|------|
| 1 | [01-upgrade-guide.md](docs/day2-operations/01-upgrade-guide.md) | 클러스터 업그레이드 |
| 2 | [upgrade-notes/](docs/day2-operations/upgrade-notes/) | 버전별 업그레이드 노트 |

### 공학적 개념 (심화 학습)

| # | 문서 | 난이도 | 설명 |
|---|------|--------|------|
| - | [engineering-concepts.md](docs/engineering-concepts.md) | - | **인덱스** - 전체 학습 가이드 |
| 00 | [개요](docs/concepts/00-overview.md) | 입문 | 학습 로드맵 및 사전 요구 지식 |
| 01 | [Linux 기초](docs/concepts/01-linux-fundamentals.md) | 중급 | 커널, 네임스페이스, cgroups |
| 02 | [컨테이너 기초](docs/concepts/02-container-fundamentals.md) | 중급 | OCI, containerd, 이미지 구조 |
| 03 | [네트워크 기초](docs/concepts/03-networking-fundamentals.md) | 중급 | OSI, TCP/IP, 라우팅, VXLAN |
| 04 | [Kubernetes 아키텍처](docs/concepts/04-kubernetes-architecture.md) | 중급~고급 | 컴포넌트, API, 서비스, 스토리지 |
| 05 | [분산 시스템](docs/concepts/05-distributed-systems.md) | 고급 | CAP, Raft, etcd, HA |
| 06 | [eBPF & Cilium](docs/concepts/06-ebpf-cilium.md) | 고급 | eBPF, Cilium, kube-proxy 대체 |
| 07 | [오프라인 인프라](docs/concepts/07-offline-infrastructure.md) | 중급~고급 | 프록시 레지스트리, 파일 서버 |

## 핵심 기술 개념

### Cilium & eBPF

Cilium은 eBPF 기반의 고성능 CNI로 kube-proxy를 완전히 대체합니다.

```
기존 (kube-proxy + iptables)         Cilium eBPF
┌─────────────┐                     ┌─────────────┐
│   Client    │                     │   Client    │
└──────┬──────┘                     └──────┬──────┘
       │                                   │
       ▼                                   │ Socket LB
┌─────────────┐                            │ (connect 시점)
│  iptables   │  O(n) 규칙                 │
│   rules     │  순차 탐색                  ▼
└──────┬──────┘                     ┌─────────────┐
       │                            │  Backend    │
       ▼                            │  Pod IP     │
┌─────────────┐                     └─────────────┘
│  Backend    │
│  Pod IP     │                     장점:
└─────────────┘                     • O(1) 해시 기반 룩업
                                    • 패킷 경로 단축
문제점:                              • conntrack 최소화
• 서비스 증가 → 규칙 증가
• 성능 저하
```

### etcd 분산 합의 (Raft)

3노드 etcd 클러스터로 고가용성을 보장합니다.

| 노드 수 | 쿼럼 | 허용 장애 | 권장 |
|---------|------|----------|------|
| 1 | 1 | 0 | 개발/테스트 |
| **3** | **2** | **1** | **프로덕션 (권장)** |
| 5 | 3 | 2 | 대규모 |

### 라우팅 모드

| 모드 | 성능 | 네트워크 요구사항 | 사용 사례 |
|------|------|------------------|----------|
| **Native** | 최적 | Pod CIDR 라우팅 필요 | 베어메탈, 동일 L2 |
| Tunnel (VXLAN) | 오버헤드 있음 | IP 연결만 필요 | 클라우드, 이기종 |

## 버전 호환성

### Kubespray ↔ Kubernetes ↔ Cilium

| Kubespray | Kubernetes | Cilium | 상태 |
|-----------|------------|--------|------|
| **v2.29.1** | 1.33.x | 1.18.4 | 최신 |
| v2.28.1 | 1.32.x | 1.17.7 | 안정 |

### 커널 요구사항

| 기능 | 최소 커널 | RHEL 10 (6.12) |
|------|----------|----------------|
| Cilium 기본 | 5.10 | 지원 |
| Netkit 데이터패스 | 6.8 | 지원 |

## 프록시 레지스트리 구성

| 포트 | 프록시 대상 | 주요 이미지 |
|------|-------------|------------|
| 5000 | docker.io | nginx, alpine 등 |
| 5001 | quay.io | cilium/cilium, cilium/operator |
| 5002 | ghcr.io | multus-cni |
| 5003 | registry.k8s.io | kube-apiserver, etcd, pause |
| 8080 | 파일 서버 | kubeadm, kubelet, kubectl 등 |

## 빠른 시작

```bash
# 1. Python 가상환경 설정
KUBESPRAY_VERSION=2.29.1
python3 -m venv venv-${KUBESPRAY_VERSION}
source venv-${KUBESPRAY_VERSION}/bin/activate

# 2. Kubespray 압축 해제 및 의존성 설치
cd sources/kubespray
tar -xzf kubespray-${KUBESPRAY_VERSION}.tar.gz
cd kubespray-${KUBESPRAY_VERSION}
pip install -U pip && pip install -r requirements.txt

# 3. 오프라인 리소스 검증
ansible-playbook ../../../playbooks/verify-offline-resources.yml

# 4. 노드 사전 설정
ansible-playbook -i inventory/offline-test/inventory.ini \
  ../../../playbooks/offline-pre-setup.yml

# 5. 클러스터 설치
ansible-playbook -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  cluster.yml

# 6. Cilium 설치 (Control Plane 노드에서)
helm install cilium ./cilium-1.18.5 \
  --namespace kube-system \
  -f cilium-values.yaml
```

## 주의사항

- Kubespray 소스는 반드시 **태그(v2.xx.y)** 기준으로 확보
- master 브랜치나 release 브랜치를 직접 사용하지 않음
- 업그레이드는 **마이너 버전을 건너뛰지 않고** 순차적으로 수행
- 자세한 내용은 [01-kubespray-version-guide.md](docs/day0-preparation/01-kubespray-version-guide.md) 참조

## 라이선스

이 프로젝트는 자유롭게 사용 및 수정이 가능합니다.
