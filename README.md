# Kubespray 폐쇄망 설치 가이드

폐쇄망(오프라인) 환경에서 Kubespray를 사용하여 Kubernetes 클러스터를 배포하고 운영하기 위한 가이드입니다.

## 프로젝트 구조

```
kubespray-offline-install-guide/
├── docs/
│   ├── day0-preparation/           # Day 0: 설계 및 준비
│   │   ├── 01-kubespray-version-guide.md
│   │   ├── 02-rhel10-install.md
│   │   ├── 03-offline-prepare.md
│   │   └── 04-proxy-container-registry.md
│   ├── day1-deployment/            # Day 1: 설치 및 배포
│   │   ├── 01-cluster-install.md
│   │   └── 02-cilium-install.md
│   └── day2-operations/            # Day 2: 운영 및 유지보수
│       ├── 01-upgrade-guide.md
│       └── upgrade-notes/
│           └── 2.28-to-2.29.md
├── configs/                        # 샘플 설정 파일
│   ├── cilium/
│   │   └── cilium-values-custom.yaml
│   └── kubespray/
│       ├── custom-config.yml
│       └── inventory.ini
├── playbooks/                      # Ansible Playbooks
│   └── offline-pre-setup.yml
└── sources/                        # 소스 파일
    ├── kubespray/
    │   ├── kubespray-2.28.1.tar.gz
    │   └── kubespray-2.29.1.tar.gz
    └── cilium/
```

## Day 0 / Day 1 / Day 2 운영 모델

| Day | 단계 | 설명 |
|-----|------|------|
| **Day 0** | 설계 및 준비 | 아키텍처 설계, 소스 확보, 환경 구성 |
| **Day 1** | 설치 및 배포 | 클러스터 설치, CNI 배포 |
| **Day 2** | 운영 및 유지보수 | 업그레이드, 노드 관리, 트러블슈팅 |

## 시작하기

### 사전 요구 사항

- RHEL 10 또는 호환 OS
- Ansible 설치
- Python 3.x

### Day 0: 준비 단계

1. **[01-kubespray-version-guide.md](docs/day0-preparation/01-kubespray-version-guide.md)** - Kubespray 버전 및 소스 확보 방법
2. **[02-rhel10-install.md](docs/day0-preparation/02-rhel10-install.md)** - RHEL 10 설치 및 Subscription 등록
3. **[03-offline-prepare.md](docs/day0-preparation/03-offline-prepare.md)** - 오프라인 설치 파일 준비
4. **[04-proxy-container-registry.md](docs/day0-preparation/04-proxy-container-registry.md)** - 프록시 컨테이너 레지스트리 구성

### Day 1: 배포 단계

5. **[01-cluster-install.md](docs/day1-deployment/01-cluster-install.md)** - Kubernetes 클러스터 설치
6. **[02-cilium-install.md](docs/day1-deployment/02-cilium-install.md)** - Cilium CNI 설치

### Day 2: 운영 단계

7. **[01-upgrade-guide.md](docs/day2-operations/01-upgrade-guide.md)** - 클러스터 업그레이드 가이드
8. **[upgrade-notes/](docs/day2-operations/upgrade-notes/)** - 버전별 업그레이드 노트

## 환경 구성 개요

| 구성 요소 | 설명 |
|-----------|------|
| Ansible 컨트롤러 | 온라인 환경에서 패키지/이미지 다운로드 |
| 프록시 레지스트리 | 컨테이너 이미지 프록시 (docker.io, quay.io, ghcr.io, registry.k8s.io) |
| 파일 서버 | Kubernetes 바이너리 파일 제공 (Nginx) |
| 클러스터 노드 | Control Plane 3대 + Worker 노드들 |

### 프록시 레지스트리 포트

| 포트 | 프록시 대상 |
|------|-------------|
| 5000 | docker.io |
| 5001 | quay.io |
| 5002 | ghcr.io |
| 5003 | registry.k8s.io |
| 8080 | 바이너리 파일 서버 |

## Kubespray 버전

| 버전 | Kubernetes | 상태 |
|------|------------|------|
| v2.29.1 | 1.33.x | 최신 |
| v2.28.1 | 1.32.x | 안정 |

## 주의사항

- Kubespray 소스는 반드시 **태그(v2.xx.y)** 기준으로 확보
- master 브랜치나 release 브랜치를 직접 사용하지 않음
- 자세한 내용은 [01-kubespray-version-guide.md](docs/day0-preparation/01-kubespray-version-guide.md) 참조

## 라이선스

이 프로젝트는 자유롭게 사용 및 수정이 가능합니다.
