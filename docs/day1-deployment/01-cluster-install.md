# Kubespray 오프라인 클러스터 설치

오프라인 환경에서 Kubespray를 사용하여 Kubernetes 클러스터를 설치한다.

**사전 요구 사항:**
- [03-kubespray-offline-prepare.md](../day0-preparation/03-kubespray-offline-prepare.md) - 오프라인 파일 준비 완료
- [05-proxy-container-registry.md](../day0-preparation/05-proxy-container-registry.md) - 프록시 레지스트리 구성 완료

## 1. Kubespray 환경 설정

### 1.1. Python 가상환경 설정

> ⚠️ **중요**: Kubespray 버전마다 Python 의존성이 다르므로, 버전별 가상환경을 설정해야 합니다.

```bash
# 작업 디렉토리 설정
KUBESPRAY_BASE=/path/to/kubespray-offline-install-guide/sources/kubespray
KUBESPRAY_VERSION=2.29.1
VENVDIR=${KUBESPRAY_BASE}/venv-${KUBESPRAY_VERSION}
KUBESPRAYDIR=${KUBESPRAY_BASE}/kubespray-${KUBESPRAY_VERSION}

# 가상환경 생성 및 활성화
python3 -m venv $VENVDIR
source $VENVDIR/bin/activate

# 의존성 설치
cd $KUBESPRAYDIR
pip install -U pip
pip install -r requirements.txt

# 설치 확인
ansible --version
```

**참고**: 버전별로 별도의 가상환경(`venv-2.28.1`, `venv-2.29.1`)을 유지하면 롤백 시에도 유용합니다.

### 1.2. 가상환경 재사용

이미 가상환경이 생성되어 있다면 활성화만 하면 됩니다:

```bash
KUBESPRAY_BASE=/root/kubespray-offline-install-guide/sources/kubespray
KUBESPRAY_VERSION=2.29.1
source ${KUBESPRAY_BASE}/venv-${KUBESPRAY_VERSION}/bin/activate
cd ${KUBESPRAY_BASE}/kubespray-${KUBESPRAY_VERSION}
```

### 1.3. 오프라인 리소스 검증 (권장)

클러스터 설치 전 파일 서버와 레지스트리가 정상적으로 준비되었는지 확인합니다:

```bash
# files.list 생성 (오프라인 환경에서도 가능)
cd contrib/offline
bash generate_list.sh

# 리소스 검증
cd $KUBESPRAYDIR
ansible-playbook ../../../playbooks/verify-offline-resources.yml
```

## 2. 노드 사전 설정

클러스터 설치 전 필요한 설정을 `offline-pre-setup.yml` Playbook으로 자동화하여 실행한다.

### 2.1. Playbook 개요

`offline-pre-setup.yml`은 다음 작업을 수행한다.

| Play | 설명 | 태그 |
|------|------|------|
| Play 1 | 필수 패키지 다운로드 (Ansible 컨트롤러) | `download` |
| Play 2 | 패키지 전송 및 설치 (대상 노드) | `packages` |
| Play 3 | DNS 설정 | `dns` |
| Play 4 | /etc/hosts 설정 | `hosts` |
| Play 5 | Swap 비활성화 | `swap` |
| Play 6 | 최종 검증 | `verify` |

### 2.2. Playbook 전체 실행

모든 사전 설정을 한 번에 실행한다.

```bash
cd /root/kubespray
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml
```

### 2.3. 변수 오버라이드

환경에 맞게 변수를 변경하여 실행할 수 있다.

```bash
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml \
  -e "repo_server_ip=192.168.108.201" \
  -e "repo_server_hostname=repo.kubespray.miribit.lab"
```

### 2.4. 개별 태그로 실행

특정 작업만 선택적으로 실행할 수 있다.

```bash
# 패키지 다운로드만
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags download

# 패키지 전송/설치만
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags packages

# DNS 설정만
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags dns

# /etc/hosts 설정만
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags hosts

# Swap 비활성화만
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags swap

# 검증만
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags verify
```

### 2.5. 검증

Playbook 실행 후 검증 태그로 설정 상태를 확인한다.

```bash
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags verify
```

정상적인 경우 다음과 같은 출력이 표시된다.

```
TASK [[VERIFY] Summary] ********************************************************
ok: [node1] => {
    "msg": [
        "=== Pre-Setup Summary for node1 ===",
        "Packages: ['conntrack-tools-1.x.x', 'ipvsadm-1.x.x', 'socat-1.x.x']",
        "DNS: nameserver 127.0.0.1",
        "Swap:               Swap:          0B          0B          0B"
    ]
}
```

## 3. Kubespray 설정

### 3.1. offline.yml 설정

파일 레포지토리 URL을 설정한다.

**파일 경로:** `inventory/offline-test/group_vars/all/offline.yml`

```yaml
---
## Global Offline settings
files_repo: "http://repo.kubespray.miribit.lab:8080"
```

### 3.2. containerd.yml 설정

컨테이너 레지스트리 미러를 설정한다.

**파일 경로:** `inventory/offline-test/group_vars/all/containerd.yml`

```yaml
---
containerd_registries_mirrors:
  - prefix: docker.io
    mirrors:
      - host: http://repo.kubespray.miribit.lab:5000
        capabilities: ["pull", "resolve"]
        skip_verify: true
  - prefix: quay.io
    mirrors:
      - host: http://repo.kubespray.miribit.lab:5001
        capabilities: ["pull", "resolve"]
        skip_verify: true
  - prefix: ghcr.io
    mirrors:
      - host: http://repo.kubespray.miribit.lab:5002
        capabilities: ["pull", "resolve"]
        skip_verify: true
  - prefix: registry.k8s.io
    mirrors:
      - host: http://repo.kubespray.miribit.lab:5003
        capabilities: ["pull", "resolve"]
        skip_verify: true
```

## 4. 클러스터 설치

### 4.1. Kubespray Playbook 실행

```bash
cd ~/kubespray

ansible-playbook -i inventory/offline-test/inventory.ini cluster.yml -f 23 -e @inventory/offline-test/custom-config.yml

# -f: ansible-playbook 동시 실행
```

> **참고:** `download_run_once=true`, `download_localhost=true` 옵션은 Ansible 컨트롤러에 nerdctl이 없으면 사용할 수 없다. 프록시 미러 환경에서는 각 노드에서 직접 pull하므로 이 옵션이 필요하지 않다.

## 5. 전체 실행 순서 요약

```bash
# 0. 환경 변수 설정 및 가상환경 활성화
KUBESPRAY_BASE=/root/kubespray-offline-install-guide/sources/kubespray
KUBESPRAY_VERSION=2.29.1
source ${KUBESPRAY_BASE}/venv-${KUBESPRAY_VERSION}/bin/activate
cd ${KUBESPRAY_BASE}/kubespray-${KUBESPRAY_VERSION}

# 1. 오프라인 리소스 검증 (권장)
cd contrib/offline && bash generate_list.sh && cd ../..
ansible-playbook ../../../playbooks/verify-offline-resources.yml

# 2. 노드 사전 준비 (패키지, DNS, /etc/hosts, Swap 설정)
ansible-playbook -i inventory/offline-test/inventory.ini \
  ../../../playbooks/offline-pre-setup.yml

# 3. 검증 (선택사항)
ansible-playbook -i inventory/offline-test/inventory.ini \
  ../../../playbooks/offline-pre-setup.yml --tags verify

# 4. 클러스터 설치
ansible-playbook -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  cluster.yml
```

## 6. 트러블슈팅

### 6.1. 에러: "download_run_once | type_debug == 'bool'"

**원인:** CLI에서 boolean 값을 문자열로 전달함

**해결:** JSON 형식 사용

```bash
# 잘못된 방법
-e download_run_once=true

# 올바른 방법
-e '{"download_run_once": true}'
```

### 6.2. 에러: "No package conntrack available"

**원인:** 오프라인 환경에서 패키지 리포지토리 없음

**해결:** 사전 준비 Playbook 실행

```bash
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags packages
```

### 6.3. 에러: "nameserver should not be empty in /etc/resolv.conf"

**원인:** DNS 설정 없음

**해결:** DNS 설정 Playbook 실행

```bash
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags dns
```

### 6.4. 에러: "No such file or directory: nerdctl"

**원인:** `download_localhost=true` 사용 시 Ansible 컨트롤러에 nerdctl 없음

**해결:** 옵션 제거하고 실행 (각 노드에서 직접 pull)

### 6.5. 에러: "running with swap on is not supported"

**원인:** Swap이 활성화되어 있음

**해결:** Swap 비활성화 Playbook 실행

```bash
ansible-playbook -i inventory/offline-test/inventory.ini offline-pre-setup.yml --tags swap
```

## 부록: 수동 실행 방법

Playbook을 사용하지 않고 수동으로 설정하는 경우 아래 명령어를 참고한다.

### A.1. 필수 패키지 다운로드 및 배포

```bash
# 패키지 다운로드 (Ansible 컨트롤러에서)
mkdir -p /tmp/offline-rpms
cd /tmp/offline-rpms
dnf download --resolve conntrack-tools ipvsadm socat

# 모든 노드에 복사 및 설치
cd /root/kubespray
ansible -i inventory/offline-test/inventory.ini all -m copy \
  -a "src=/tmp/offline-rpms/ dest=/tmp/offline-rpms/" --become

ansible -i inventory/offline-test/inventory.ini all -m shell \
  -a "dnf install -y /tmp/offline-rpms/*.rpm" --become
```

### A.2. DNS 설정

```bash
ansible -i inventory/offline-test/inventory.ini all -m lineinfile \
  -a "path=/etc/resolv.conf line='nameserver 127.0.0.1' create=yes" --become
```

### A.3. /etc/hosts 설정

```bash
# 레포지토리 서버 추가
ansible -i inventory/offline-test/inventory.ini all -m lineinfile \
  -a "path=/etc/hosts line='192.168.108.201 repo.kubespray.miribit.lab'" --become
```

### A.4. Swap 비활성화

```bash
# Swap 즉시 비활성화
ansible -i inventory/offline-test/inventory.ini all -m command \
  -a "swapoff -a" --become

# /etc/fstab에서 swap 항목 제거
ansible -i inventory/offline-test/inventory.ini all -m lineinfile \
  -a "path=/etc/fstab regexp='.*swap.*' state=absent" --become
```
