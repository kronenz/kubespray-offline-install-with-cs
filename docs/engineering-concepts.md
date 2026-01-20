# Kubespray 폐쇄망 설치를 위한 공학적 개념 가이드

이 문서는 폐쇄망(Air-Gapped) 환경에서 Kubespray를 사용하여 Kubernetes 클러스터를 배포할 때 필요한 공학적 개념에 대한 **학습 가이드 인덱스**입니다.

> **상세 학습 문서**: 각 주제에 대한 심층적인 내용은 [concepts/](./concepts/) 디렉토리의 개별 문서를 참조하세요.

---

## 학습 로드맵

```
                           Kubespray 폐쇄망 설치 학습 경로
                           
                    ┌─────────────────────────────────────┐
                    │         00. 학습 개요               │
                    │      (개념 문서 로드맵)              │
                    └─────────────────┬───────────────────┘
                                      │
            ┌─────────────────────────┼─────────────────────────┐
            │                         │                         │
            ▼                         ▼                         ▼
   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
   │  01. Linux      │    │  02. 컨테이너    │    │  03. 네트워크   │
   │     기초        │    │     기초        │    │     기초        │
   │                 │    │                 │    │                 │
   │ • 커널          │    │ • OCI 표준      │    │ • OSI/TCP-IP    │
   │ • 네임스페이스   │    │ • containerd   │    │ • 라우팅        │
   │ • cgroups       │    │ • 이미지 구조   │    │ • VXLAN         │
   └────────┬────────┘    └────────┬────────┘    └────────┬────────┘
            │                      │                      │
            └──────────────────────┼──────────────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────────────┐
                    │     04. Kubernetes 아키텍처         │
                    │                                     │
                    │  • Control Plane / Node 컴포넌트    │
                    │  • Pod 생성 흐름                    │
                    │  • Service / Ingress               │
                    └─────────────────┬───────────────────┘
                                      │
            ┌─────────────────────────┼─────────────────────────┐
            │                         │                         │
            ▼                         ▼                         ▼
   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
   │  05. 분산       │    │  06. eBPF &     │    │  07. 오프라인   │
   │     시스템      │    │     Cilium      │    │     인프라      │
   │                 │    │                 │    │                 │
   │ • CAP 이론      │    │ • eBPF 아키텍처 │    │ • 프록시 레지스트리│
   │ • Raft 알고리즘 │    │ • Cilium 컴포넌트│   │ • 파일 서버     │
   │ • etcd 클러스터 │    │ • kube-proxy 대체│   │ • 버전 관리     │
   └─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## 문서 목록

### 기초 (Foundation)

| # | 문서 | 난이도 | 설명 |
|---|------|--------|------|
| 00 | [학습 개요](./concepts/00-overview.md) | 입문 | 전체 학습 로드맵 및 사전 요구 지식 |
| 01 | [Linux 기초](./concepts/01-linux-fundamentals.md) | 중급 | 커널, 네임스페이스, cgroups, 파일시스템 |
| 02 | [컨테이너 기초](./concepts/02-container-fundamentals.md) | 중급 | OCI 표준, 런타임, 이미지 구조, containerd |
| 03 | [네트워크 기초](./concepts/03-networking-fundamentals.md) | 중급 | OSI, TCP/IP, 라우팅, NAT, VXLAN |

### 심화 (Advanced)

| # | 문서 | 난이도 | 설명 |
|---|------|--------|------|
| 04 | [Kubernetes 아키텍처](./concepts/04-kubernetes-architecture.md) | 중급~고급 | 컴포넌트, API, 워크로드, 서비스, 스토리지 |
| 05 | [분산 시스템](./concepts/05-distributed-systems.md) | 고급 | CAP, Raft, etcd, 고가용성 설계 |
| 06 | [eBPF와 Cilium](./concepts/06-ebpf-cilium.md) | 고급 | eBPF 아키텍처, Cilium CNI, kube-proxy 대체 |

### 실습 (Practice)

| # | 문서 | 난이도 | 설명 |
|---|------|--------|------|
| 07 | [오프라인 인프라](./concepts/07-offline-infrastructure.md) | 중급~고급 | 프록시 레지스트리, 파일 서버, 버전 관리 |

---

## 빠른 참조

### 프로젝트 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           폐쇄망 (Air-Gapped Network)                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────┐       ┌──────────────────────────────────────┐   │
│  │ Ansible 컨트롤러  │       │           Kubernetes 클러스터         │   │
│  │                  │  SSH  │  ┌────────────────────────────────┐  │   │
│  │  - Kubespray     │──────►│  │      Control Plane (x3, HA)   │  │   │
│  │  - Python venv   │       │  │   kube-apiserver + etcd (Raft) │  │   │
│  └──────────────────┘       │  └────────────────────────────────┘  │   │
│           │                 │  ┌────────────────────────────────┐  │   │
│           │                 │  │       Worker Nodes (x20)       │  │   │
│           │                 │  │   kubelet + containerd + Cilium │  │   │
│           │                 │  └────────────────────────────────┘  │   │
│           │                 └──────────────────────────────────────┘   │
│           │                                    ▲                       │
│           ▼                                    │ Pull                  │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │                 레포지토리 서버 (Repo Server)                    │   │
│  │  ┌────────────────────────┐  ┌─────────────────────────────┐  │   │
│  │  │   프록시 레지스트리     │  │       파일 서버 (Nginx)       │  │   │
│  │  │  :5000 docker.io       │  │  :8080 바이너리 파일          │  │   │
│  │  │  :5001 quay.io         │  │   ├─ kubeadm, kubelet        │  │   │
│  │  │  :5002 ghcr.io         │  │   ├─ etcd, containerd        │  │   │
│  │  │  :5003 registry.k8s.io │  │   └─ cni-plugins, helm       │  │   │
│  │  └────────────────────────┘  └─────────────────────────────┘  │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 핵심 기술 스택

| 구성 요소 | 기술 | 학습 문서 |
|-----------|------|----------|
| Container Runtime | containerd | [02-container-fundamentals.md](./concepts/02-container-fundamentals.md) |
| CNI | Cilium (eBPF) | [06-ebpf-cilium.md](./concepts/06-ebpf-cilium.md) |
| Distributed Storage | etcd (Raft) | [05-distributed-systems.md](./concepts/05-distributed-systems.md) |
| Cluster Provisioning | Kubespray (Ansible) | [07-offline-infrastructure.md](./concepts/07-offline-infrastructure.md) |
| Registry Proxy | Registry:2 | [07-offline-infrastructure.md](./concepts/07-offline-infrastructure.md) |

### 네트워크 설계

| 네트워크 | CIDR | 용도 | 학습 문서 |
|----------|------|------|----------|
| 물리 네트워크 | 192.168.108.0/24 | 노드 통신 | [03-networking-fundamentals.md](./concepts/03-networking-fundamentals.md) |
| Pod 네트워크 | 10.200.0.0/16 | Cilium IPAM | [06-ebpf-cilium.md](./concepts/06-ebpf-cilium.md) |
| Service 네트워크 | 10.201.0.0/18 | ClusterIP | [04-kubernetes-architecture.md](./concepts/04-kubernetes-architecture.md) |

### 버전 호환성

| Kubespray | Kubernetes | Cilium | 커널 요구사항 |
|-----------|------------|--------|--------------|
| v2.29.1 | 1.33.x | 1.18.x | 5.10+ (netkit: 6.7+) |
| v2.28.1 | 1.32.x | 1.17.x | 5.10+ |

---

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

| 단계 | 학습 문서 | 실습 문서 |
|------|----------|----------|
| Day 0 | 모든 concepts/ 문서 | [day0-preparation/](./day0-preparation/) |
| Day 1 | [04](./concepts/04-kubernetes-architecture.md), [06](./concepts/06-ebpf-cilium.md), [07](./concepts/07-offline-infrastructure.md) | [day1-deployment/](./day1-deployment/) |
| Day 2 | [05](./concepts/05-distributed-systems.md) (etcd 백업/복구) | [day2-operations/](./day2-operations/) |

---

## 학습 권장 순서

### 입문자 (Kubernetes 경험 없음)

1. [00-overview.md](./concepts/00-overview.md) - 전체 구조 파악
2. [01-linux-fundamentals.md](./concepts/01-linux-fundamentals.md) - 컨테이너의 기반 이해
3. [02-container-fundamentals.md](./concepts/02-container-fundamentals.md) - 컨테이너 기술 이해
4. [03-networking-fundamentals.md](./concepts/03-networking-fundamentals.md) - 네트워크 기초
5. [04-kubernetes-architecture.md](./concepts/04-kubernetes-architecture.md) - Kubernetes 핵심
6. 나머지 문서 순차적으로

### 중급자 (Kubernetes 운영 경험 있음)

1. [00-overview.md](./concepts/00-overview.md) - 학습 범위 확인
2. [05-distributed-systems.md](./concepts/05-distributed-systems.md) - etcd/HA 심화
3. [06-ebpf-cilium.md](./concepts/06-ebpf-cilium.md) - Cilium 심화
4. [07-offline-infrastructure.md](./concepts/07-offline-infrastructure.md) - 오프라인 구축

### 특정 주제만 필요한 경우

- **네트워킹**: 03 → 06
- **고가용성**: 05 (Raft, etcd)
- **오프라인 구축**: 07

---

## 참고 자료

### 공식 문서

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubespray Documentation](https://kubespray.io/)
- [Cilium Documentation](https://docs.cilium.io/)
- [eBPF.io](https://ebpf.io/)
- [containerd Documentation](https://containerd.io/docs/)

### 프로젝트 내 문서

- [Day 0: 준비 단계](./day0-preparation/)
- [Day 1: 배포 단계](./day1-deployment/)
- [Day 2: 운영 단계](./day2-operations/)
- [설정 파일 샘플](../configs/)
- [Ansible Playbooks](../playbooks/)

---

> **Note**: 이 문서들은 프로젝트의 실제 구성 (RHEL 10, Kubespray v2.29.1, Cilium 1.18.x)을 기준으로 작성되었습니다.
