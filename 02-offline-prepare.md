# Offline 설치 파일 준비

Online 연결이 되어있는 Node에서 오프라인 설치에 필요한 파일을 준비한다.

**참고 문서:**
- [Kubespray Offline 가이드](https://github.com/kubernetes-sigs/kubespray/blob/master/contrib/offline/README.md)
- [Ansible 설치 가이드](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ansible/ansible.md#installing-ansible)

## 1. Kubespray 및 Ansible 환경 구성

실행할 쉘 파일이 Ansible로 실행되므로 Ansible 배포가 선행되어야 한다.

### 1.1. Kubespray 클론

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
```

### 1.2. Python 가상환경 설정 및 의존성 설치

```bash
VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
python3 -m venv $VENVDIR
source $VENVDIR/bin/activate
cd $KUBESPRAYDIR
pip install -r requirements.txt
```

## 2. 파일 및 이미지 목록 생성

현재 Kubespray 버전에 필요한 파일과 이미지 목록을 추출하는 스크립트를 실행한다.

```bash
cd contrib/offline
bash generate_list.sh
```

`kubespray/contrib/offline/temp` 디렉토리에 아래 파일이 생성된다.

```
ls -l temp/
total 16
-rw-r--r--. 1 root root 2024 Dec  3 19:07 files.list
-rw-r--r--. 1 root root 2860 Dec  3 19:07 files.list.template
-rw-r--r--. 1 root root 2281 Dec  3 19:07 images.list
-rw-r--r--. 1 root root 3151 Dec  3 19:07 images.list.template
```

## 3. 오프라인 파일 다운로드

### 3.1. 환경 구성 개요

| 구성 요소 | 설정 방식 |
|-----------|----------|
| Container Registry | 프록시 설정 |
| File Repository | 실제 파일 제공 |

본 가이드에서는 파일만 다운로드하여 압축한다.

### 3.2. 파일 다운로드 및 압축

```bash
export FILES_LIST=temp/files.list
NO_HTTP_SERVER=true bash manage-offline-files.sh
```

`offline-files.tar.gz` 파일이 생성되었는지 확인한다.

## 4. 레포 서버로 파일 전송

### 4.1. 파일 전송

```bash
scp offline-files.tar.gz root@<repo-server-ip>:/root
scp nginx.conf root@<repo-server-ip>:/root
```

### 4.2. 레포 서버에서 압축 해제

```bash
ssh root@<repo-server-ip>
tar -xvf offline-files.tar.gz
```

압축 해제 후 디렉토리 구조:

```
ls -al offline-files
total 8
drwxr-xr-x.  6 root root   90 Dec  3 19:24 .
dr-xr-x---.  4 root root 4096 Dec  3 19:31 ..
drwxr-xr-x.  3 root root   21 Dec  3 19:24 dl.k8s.io
drwxr-xr-x.  2 root root   45 Dec  3 19:24 get.helm.sh
drwxr-xr-x. 16 root root 4096 Dec  3 19:24 github.com
drwxr-xr-x.  4 root root   33 Dec  3 19:24 storage.googleapis.com
```

## 5. Nginx 파일 서버 구동

Nginx 컨테이너를 사용하여 파일 레포지토리를 구성한다.

```bash
sudo podman run --restart=always -d -p 8080:80 \
  --name nginx \
  -v $(pwd)/offline-files:/usr/share/nginx/html/download:z \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:z \
  nginx:alpine
```

## 6. 확인

`http://<repo-server-ip>:8080/`에 접속하여 파일 다운로드가 가능한지 확인한다.
