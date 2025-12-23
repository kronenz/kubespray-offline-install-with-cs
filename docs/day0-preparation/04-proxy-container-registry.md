# 프록시 컨테이너 레지스트리 구성

레포지토리 서버용 VM에 프록시 컨테이너를 구동하여 내부 환경에서 외부 컨테이너 레지스트리에 접근할 수 있도록 구성한다.

## 1. 프록시 포트 구성

각 레지스트리별 프록시 포트는 다음과 같다.

| 레지스트리 | 포트 |
|-----------|------|
| docker.io | 5000 |
| quay.io | 5001 |
| ghcr.io | 5002 |
| registry.k8s.io | 5003 |

## 2. 사전 준비

### 2.1. SELinux 설정 변경

프록시 컨테이너 구동을 위해 SELinux를 비활성화한다.

```bash
setenforce 0
```

### 2.2. 프록시 디렉토리 생성

프록시 설정 파일을 저장할 디렉토리를 생성한다.

```bash
mkdir -p /root/proxies/{docker,quay,ghcr,k8s}
```

## 3. 레지스트리 설정 파일 구성

### 3.1. registries.conf 파일 수정

`/etc/containers/registries.conf` 파일 하단에 프록시 레지스트리 설정을 추가한다.

```bash
cat <<EOF >> /etc/containers/registries.conf
# Docker Hub Proxy
[[registry]]
location = "repo.kubespray.miribit.lab:5000"
insecure = true

# Quay.io Proxy
[[registry]]
location = "repo.kubespray.miribit.lab:5001"
insecure = true

# GHCR Proxy
[[registry]]
location = "repo.kubespray.miribit.lab:5002"
insecure = true

# K8s Registry Proxy
[[registry]]
location = "repo.kubespray.miribit.lab:5003"
insecure = true

# Localhost testing (for all ports)
[[registry]]
location = "localhost:5000"
insecure = true

[[registry]]
location = "localhost:5001"
insecure = true

[[registry]]
location = "localhost:5002"
insecure = true

[[registry]]
location = "localhost:5003"
insecure = true
EOF
```

### 3.2. Docker Hub 프록시 설정 파일 생성

포트 5000번을 사용하는 Docker Hub 프록시 설정 파일을 생성한다.

```bash
cat <<EOF > /root/proxies/docker/config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
proxy:
  remoteurl: https://registry-1.docker.io
EOF
```

### 3.3. Quay.io 프록시 설정 파일 생성

포트 5001번을 사용하는 Quay.io 프록시 설정 파일을 생성한다.

```bash
cat <<EOF > /root/proxies/quay/config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5001
  headers:
    X-Content-Type-Options: [nosniff]
proxy:
  remoteurl: https://quay.io
EOF
```

### 3.4. GHCR 프록시 설정 파일 생성

포트 5002번을 사용하는 GHCR 프록시 설정 파일을 생성한다.

```bash
cat <<EOF > /root/proxies/ghcr/config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5002
  headers:
    X-Content-Type-Options: [nosniff]
proxy:
  remoteurl: https://ghcr.io
EOF
```

### 3.5. Registry.k8s.io 프록시 설정 파일 생성

포트 5003번을 사용하는 Registry.k8s.io 프록시 설정 파일을 생성한다.

```bash
cat <<EOF > /root/proxies/k8s/config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5003
  headers:
    X-Content-Type-Options: [nosniff]
proxy:
  remoteurl: https://registry.k8s.io
EOF
```

## 4. 레지스트리 프록시 컨테이너 실행

### 4.1. Docker Hub 프록시 컨테이너 실행

```bash
sudo podman run -d --name proxy-docker -p 5000:5000 --restart=always \
  -v /root/proxies/docker/config.yml:/etc/docker/registry/config.yml:z \
  registry:2
```

### 4.2. Quay 프록시 컨테이너 실행

```bash
sudo podman run -d --name proxy-quay -p 5001:5001 --restart=always \
  -v /root/proxies/quay/config.yml:/etc/docker/registry/config.yml:z \
  registry:2
```

### 4.3. GHCR 프록시 컨테이너 실행

```bash
sudo podman run -d --name proxy-ghcr -p 5002:5002 --restart=always \
  -v /root/proxies/ghcr/config.yml:/etc/docker/registry/config.yml:z \
  registry:2
```

### 4.4. K8s Registry 프록시 컨테이너 실행

```bash
sudo podman run -d --name proxy-k8s -p 5003:5003 --restart=always \
  -v /root/proxies/k8s/config.yml:/etc/docker/registry/config.yml:z \
  registry:2
```

## 5. 확인

### 5.1. 컨테이너 실행 상태 확인

모든 프록시 컨테이너가 정상적으로 실행 중인지 확인한다.

```bash
sudo podman ps
```

정상적으로 실행 중인 경우 다음과 같은 출력이 표시된다.

```
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS         PORTS                             NAMES
e4573ca404d1  docker.io/library/nginx:alpine  nginx -g daemon o...  4 hours ago     Up 4 hours     0.0.0.0:8080->80/tcp              nginx
d2bd36062d53  docker.io/library/registry:2    /etc/docker/regis...  37 minutes ago  Up 16 minutes  0.0.0.0:5000->5000/tcp            proxy-docker
39b4b6d74ed9  docker.io/library/registry:2    /etc/docker/regis...  37 minutes ago  Up 16 minutes  0.0.0.0:5001->5001/tcp, 5000/tcp  proxy-quay
5bb29eea4e3b  docker.io/library/registry:2    /etc/docker/regis...  37 minutes ago  Up 16 minutes  0.0.0.0:5002->5002/tcp, 5000/tcp  proxy-ghcr
9899aa16fa2d  docker.io/library/registry:2    /etc/docker/regis...  37 minutes ago  Up 16 minutes  0.0.0.0:5003->5003/tcp, 5000/tcp  proxy-k8s
```

### 5.2. 프록시 이미지 Pull 테스트

다른 서버에서 프록시를 통해 컨테이너 이미지를 pull할 수 있는지 확인한다.

#### 5.2.1. /etc/hosts 파일 설정

테스트 서버의 `/etc/hosts` 파일에 레포지토리 서버 호스트명을 추가한다.

```bash
echo "<repo-server-ip>  repo.kubespray.miribit.lab" >> /etc/hosts
```

**참고:** 호스트 이름은 실제 환경에 맞게 변경한다.

#### 5.2.2. Docker Hub 프록시 테스트

Docker Hub의 이미지를 프록시를 통해 pull한다.

```bash
# 원본: docker.io/library/alpine:latest
# 프록시: repo.kubespray.miribit.lab:5000/library/alpine:latest
podman pull --tls-verify=false repo.kubespray.miribit.lab:5000/library/alpine:latest
```

#### 5.2.3. Quay.io 프록시 테스트

Quay.io의 이미지를 프록시를 통해 pull한다.

```bash
# 원본: quay.io/prometheus/busybox:latest
# 프록시: repo.kubespray.miribit.lab:5001/prometheus/busybox:latest
podman pull --tls-verify=false repo.kubespray.miribit.lab:5001/prometheus/busybox:latest
```

#### 5.2.4. GHCR 프록시 테스트

GHCR의 이미지를 프록시를 통해 pull한다.

```bash
# 원본: ghcr.io/k8snetworkplumbingwg/multus-cni:v4.2.2
# 프록시: repo.kubespray.miribit.lab:5002/k8snetworkplumbingwg/multus-cni:v4.2.2
podman pull --tls-verify=false repo.kubespray.miribit.lab:5002/k8snetworkplumbingwg/multus-cni:v4.2.2
```

#### 5.2.5. Registry.k8s.io 프록시 테스트

Registry.k8s.io의 이미지를 프록시를 통해 pull한다.

```bash
# 원본: registry.k8s.io/pause:3.9
# 프록시: repo.kubespray.miribit.lab:5003/pause:3.9
podman pull --tls-verify=false repo.kubespray.miribit.lab:5003/pause:3.9
```

