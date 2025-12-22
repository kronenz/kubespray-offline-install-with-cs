# Kubespray 폐쇄망 설치 가이드

폐쇄망(오프라인) 환경에서 Kubespray를 사용하여 Kubernetes 클러스터를 배포하기 위한 가이드 및 설정 파일을 제공합니다.

## 프로젝트 구조

```
kubespray-offline-install-guide/
├── docs/                           # 설치 가이드 문서
│   ├── 00-kubespray-version-guide.md   # Kubespray 버전 관리 가이드
│   ├── 01-rhel10-install.md            # RHEL 10 설치
│   ├── 02-offline-prepare.md           # 오프라인 파일 준비
│   ├── 03-proxy-container-registry.md  # 프록시 레지스트리 구성
│   ├── 04-kubespray-cluster-install.md # 클러스터 설치
│   └── 05-cilium-install.md            # Cilium CNI 설치
├── configs/                        # 샘플 설정 파일
│   ├── cilium/
│   │   └── cilium-values-custom.yaml   # Cilium Helm values
│   └── kubespray/
│       ├── custom-config.yml           # Kubespray 커스텀 설정
│       └── inventory.ini               # Ansible 인벤토리 예시
├── playbooks/                      # Ansible Playbooks
│   └── offline-pre-setup.yml           # 사전 설정 자동화
└── sources/                        # 소스 파일
    ├── kubespray/
    │   ├── kubespray-2.28.1.tar.gz
    │   └── kubespray-2.29.1.tar.gz
    └── cilium/
        └── .gitkeep
```

## 시작하기

### 사전 요구 사항

- RHEL 10 또는 호환 OS
- Ansible 설치
- Python 3.x

### 설치 순서

문서를 순서대로 따라하시면 됩니다:

1. **[00-kubespray-version-guide.md](docs/00-kubespray-version-guide.md)** - Kubespray 버전 및 소스 확보 방법
2. **[01-rhel10-install.md](docs/01-rhel10-install.md)** - RHEL 10 설치 및 Subscription 등록
3. **[02-offline-prepare.md](docs/02-offline-prepare.md)** - 오프라인 설치 파일 준비
4. **[03-proxy-container-registry.md](docs/03-proxy-container-registry.md)** - 프록시 컨테이너 레지스트리 구성
5. **[04-kubespray-cluster-install.md](docs/04-kubespray-cluster-install.md)** - Kubernetes 클러스터 설치
6. **[05-cilium-install.md](docs/05-cilium-install.md)** - Cilium CNI 설치

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

## 설정 파일 사용법

### Kubespray 설정

`configs/kubespray/custom-config.yml` 파일을 Kubespray의 `inventory/<your-cluster>/group_vars/all/` 디렉토리에 복사하여 사용합니다.

```bash
cp configs/kubespray/custom-config.yml /path/to/kubespray/inventory/<cluster>/group_vars/all/
```

### Ansible 사전 설정

클러스터 설치 전 필요한 사전 설정을 자동화합니다:

```bash
cd /path/to/kubespray
ansible-playbook -i inventory/<cluster>/inventory.ini \
  /path/to/this/repo/playbooks/offline-pre-setup.yml
```

## Kubespray 버전

| 버전 | Kubernetes | 상태 |
|------|------------|------|
| v2.29.1 | 1.32.x | 최신 |
| v2.28.1 | 1.31.x | 안정 |

## 주의사항

- Kubespray 소스는 반드시 **태그(v2.xx.y)** 기준으로 확보
- master 브랜치나 release 브랜치를 직접 사용하지 않음
- 자세한 내용은 [00-kubespray-version-guide.md](docs/00-kubespray-version-guide.md) 참조

## 라이선스

이 프로젝트는 자유롭게 사용 및 수정이 가능합니다.

