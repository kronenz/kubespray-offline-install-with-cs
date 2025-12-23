# Cilium 설치 가이드

오프라인 환경에서 Cilium CNI를 설치한다.

**사전 요구 사항:**
- [04-kubespray-cluster-install.md](04-kubespray-cluster-install.md) - Kubernetes 클러스터 설치 완료
- kubectl이 설치되어 있고 클러스터에 접근 가능한 노드

## 1. Cilium CLI 설치

Cilium CLI는 Cilium 설치 및 관리를 위한 명령줄 도구이다.

### 1.1. 온라인 환경에서 파일 다운로드

온라인 환경이 있는 노드에서 Cilium CLI 바이너리를 다운로드한다.

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
```

### 1.2. 파일 전송

다운로드한 파일을 kubectl을 사용할 노드로 전송한다.

```bash
scp cilium-linux-${CLI_ARCH}.tar.gz root@<target-node-ip>:/root
scp cilium-linux-${CLI_ARCH}.tar.gz.sha256sum root@<target-node-ip>:/root
```

**참고:** `<target-node-ip>`는 실제 대상 노드의 IP 주소로 변경한다.

### 1.3. 바이너리 설치

대상 노드에 접속하여 압축을 해제하고 바이너리를 설치한다.

```bash
ssh root@<target-node-ip>
cd /root
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm -f cilium-linux-${CLI_ARCH}.tar.gz cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
```

### 1.4. 설치 확인

Cilium CLI가 정상적으로 설치되었는지 확인한다.

```bash
cilium version --client
```

정상적인 경우 다음과 같은 출력이 표시된다.

```
cilium-cli: v0.16.x
cilium image (default): v1.15.x
```

## 2. Cilium Helm Chart 설치

Helm을 사용하여 Cilium을 설치한다.

### 2.1. Helm Chart 다운로드 및 압축 해제

온라인 환경이 있는 노드에서 Cilium Helm Chart를 GitHub에서 직접 다운로드하고 압축을 해제한다.

```bash
CILIUM_VERSION=v1.18.4
wget https://github.com/cilium/cilium/archive/refs/tags/${CILIUM_VERSION}.tar.gz
tar xzvf ${CILIUM_VERSION}.tar.gz
```

**참고:** `CILIUM_VERSION`은 설치할 Cilium 버전으로 변경한다. 최신 버전은 [Cilium Releases](https://github.com/cilium/cilium/releases)에서 확인할 수 있다.

### 2.2. Helm Chart 전송

압축 해제된 Helm Chart 디렉토리와 커스텀 values 파일을 대상 노드로 전송한다.

```bash
scp -r cilium-${CILIUM_VERSION}/ root@<target-node-ip>:/root
scp cilium-values-custom.yaml root@<target-node-ip>:/root
```

**참고:** `<target-node-ip>`는 실제 대상 노드의 IP 주소로 변경한다.

### 2.3. Values 파일 확인

커스텀 values 파일의 설정을 확인하고 필요시 수정한다.

**파일 경로:** `/root/cilium-values-custom.yaml`

주요 설정 항목:
- Kubernetes API 서버 설정
- 네트워크 라우팅 모드
- IPAM 설정
- Hubble 관찰성 설정
- Ingress Controller 설정

**참고:** `cilium-values-custom.yaml` 파일의 상세 설정은 파일 내 주석을 참고한다.

### 2.4. Cilium 설치

대상 노드에서 Helm을 사용하여 Cilium을 설치한다.

```bash
ssh root@<target-node-ip>
cd /root
helm install cilium ./cilium-${CILIUM_VERSION} -n kube-system -f cilium-values-custom.yaml
```

**참고:** `CILIUM_VERSION`은 다운로드한 버전과 동일하게 설정한다.

### 2.5. 설치 확인

Cilium이 정상적으로 설치되었는지 확인한다.

```bash
# Cilium Pod 상태 확인
kubectl get pods -n kube-system -l k8s-app=cilium

# Cilium 상태 확인
cilium status

# Cilium 연결성 테스트
cilium connectivity test
```

정상적인 경우 Cilium Pod들이 모두 `Running` 상태여야 한다.

```
NAME           READY   STATUS    RESTARTS   AGE
cilium-xxxxx   1/1     Running   0          5m
```

## 3. 확인

### 3.1. Cilium Agent 상태 확인

모든 노드에서 Cilium Agent가 정상적으로 실행 중인지 확인한다.

```bash
cilium status
```

### 3.2. 네트워크 정책 테스트

간단한 Pod를 생성하여 네트워크 연결을 테스트한다.

```bash
kubectl run test-pod --image=alpine --rm -it --restart=Never -- sh
```

Pod 내부에서 네트워크 연결을 테스트한다.

```bash
# DNS 확인
nslookup kubernetes.default.svc.cluster.local

# 외부 연결 테스트
ping -c 3 8.8.8.8
```

### 3.3. Hubble UI 접근 확인

Hubble UI가 활성화된 경우 접근 가능한지 확인한다.

```bash
# NodePort를 통한 접근
kubectl get svc -n kube-system hubble-ui

# Ingress를 통한 접근 (설정된 경우)
kubectl get ingress -n kube-system
```

**참고:** Hubble UI는 NodePort 31235 또는 Ingress를 통해 접근할 수 있다.

## 4. 트러블슈팅

### 4.1. 에러: "Cilium pods are not ready"

**원인:** Cilium Pod가 정상적으로 시작되지 않음

**해결:** Pod 로그 확인

```bash
kubectl logs -n kube-system -l k8s-app=cilium
```

### 4.2. 에러: "Unable to connect to API server"

**원인:** Kubernetes API 서버 연결 실패

**해결:** `cilium-values-custom.yaml`의 `k8sServiceHost`와 `k8sServicePort` 설정 확인

```bash
kubectl get endpoints kubernetes -n default
```

### 4.3. 에러: "IPAM allocation failed"

**원인:** Pod CIDR 범위 부족 또는 설정 오류

**해결:** `cilium-values-custom.yaml`의 IPAM 설정 확인

```bash
# 현재 Pod CIDR 확인
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
```

### 4.4. 에러: "BGP peer not established"

**원인:** BGP Control Plane 설정 오류 또는 외부 라우터 연결 실패

**해결:** BGP 설정 확인 및 외부 라우터 연결 상태 확인

```bash
# BGP 피어 상태 확인
cilium bgp peers list
```
