# RHEL 10 설치 가이드

## 1. RHEL 10 ISO 다운로드

1. [Red Hat Developers](https://developers.redhat.com/)에 접속
2. 회원 가입 후 로그인
3. 아래 경로에서 ISO 다운로드
   - https://developers.redhat.com/content-gateway/file/rhel/Red_Hat_Enterprise_Linux_10.0/rhel-10.0-x86_64-dvd.iso

## 2. 가상머신 생성 및 OS 설치

1. RHEL 가상머신 생성
2. 다운로드한 ISO로 OS 설치 진행

## 3. Red Hat Subscription 등록

### 3.1. Activation Key 생성

1. [Registration Assistant](https://console.redhat.com/insights/registration)에 접속
2. OS에서 **RHEL 9 or later** 선택
3. 화면에 표시되는 Activation Key와 Organization ID 확인

### 3.2. RHEL 서버에서 등록 명령 실행

```bash
rhc connect --activation-key <your-activation-key> --organization <your-organization>
```

### 3.3. 추가 패키지 설치

```bash
dnf install -y rhc-worker-playbook
```

## 4. 등록 확인

[Red Hat Hybrid Cloud Console](https://console.redhat.com/)에 접속하여 **Inventory > Systems**에서 등록된 호스트 확인
