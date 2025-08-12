# Security
## Kubernetes Security Primitives
1. 보안의 중요성
- 쿠버네티스는 애플리케이션을 호스팅하는 플랫폼이므로 보안이 필수적임.
- 물리적·가상 인프라와 호스트 자체를 안전하게 보호해야 함.
  - 루트 접근 차단, 암호 기반 인증 해제, SSH 키 기반 인증 사용 등.

2. API 서버 보안
- 클러스터 작업의 중심은 API 서버이며, 여기에 접근 가능한 사람을 제한하는 것이 1차 방어선.
- 인증(Authentication): 사용자/서비스 계정 식별
  - 방법: 사용자 ID·암호, 토큰, 인증서, LDAP 등 외부 인증, 서비스 계정
- 권한 부여(Authorization): 접근 권한 설정
  - 방법: RBAC(역할 기반 접근 제어), ABAC, 노드 인증자, 웹훅 등

3. 클러스터 내부 통신 보안
- API 서버, 컨트롤러 매니저, 스케줄러, etcd, Kubelet, kube-proxy 간 통신은 TLS 암호화로 보호.
- 각 구성 요소 간 인증서 설정 필요.

4. 애플리케이션 간 통신 제어
- 기본적으로 모든 파드 간 통신 가능.
- 네트워크 정책(Network Policy)으로 파드 간 접근 제한 가능.

## Authentication
### 인증의 목적
- 클러스터 보안은 내부 구성 요소 간 통신 보안과 인증·승인 메커니즘으로 관리 액세스를 보호하는 것에서 시작됨.
- 엔드유저(애플리케이션 접근)는 제외하고, 관리자·개발자·시스템/서비스 계정의 접근 제어가 주 대상.

### 쿠버네티스 사용자 계정 특징
- 쿠버네티스는 사용자 계정을 직접 관리하지 않음.
  - 사용자 정보는 외부 소스(파일, LDAP, Kerberos 등)에서 관리.
- 대신 서비스 계정(Service Account) 은 쿠버네티스에서 직접 생성·관리 가능.

### API 서버의 인증 흐름
- 모든 요청은 Kube API Server를 거치며, 작업 전 인증 수행.
- 인증 방식 예시
	- 고정 비밀번호 파일 (CSV: 사용자명, 비밀번호, 사용자 ID, [그룹])
	- 고정 토큰 파일 (비밀번호 대신 토큰 사용)
	- 인증서 기반 인증
	- 외부 인증 서비스 (LDAP, Kerberos 등)

### 고정 파일 기반 인증 특징
- CSV/토큰 파일을 API 서버에 옵션으로 지정 후 재시작해야 적용됨.
- kubeadm 사용 시 API 서버 Pod 정의 파일 수정 → 자동 재시작.
- 이 방법은 자격 증명이 평문 저장되므로 보안에 취약 → 학습·실습용으로만 권장.

## TLS Basics
### TLS 인증서의 역할
- 서버와 클라이언트 간 통신을 암호화해 데이터 도청·변조 방지
- 서버의 신원을 검증해 상호 신뢰 보장
- HTTPS(웹 서버), SSH(서버 접근) 등에서 활용

### 암호화 방식
- 대칭 암호화(Symmetric)
  - 하나의 키로 암·복호화
  - 키 교환 과정에서 키가 유출될 위험 존재
- 비대칭 암호화(Asymmetric)
  - 공개키(Public Key)와 개인키(Private Key) 쌍 사용
  - 공개키로 암호화한 데이터는 개인키로만 복호화 가능, 그 반대도 마찬가지

### SSH와 키 페어
- 비밀번호 대신 공개/개인 키로 서버 접근
- 서버의 `authorized_keys`에 공개키 등록 → 개인키를 가진 사용자만 접근 가능

### TLS 통신 과정 (HTTPS 예시)
1. 서버가 공개키를 포함한 인증서를 브라우저에 전송
2. 브라우저는 공개키로 대칭키를 암호화하여 서버로 전달
3. 서버는 개인키로 대칭키 복호화
4. 이후 대칭키로 모든 데이터 암호화 전송

### 인증서와 신뢰 체계
- 인증서에는 서버 정보, 공개키, 발급자(CA) 정보 포함
- 자가 서명(Self-signed) 인증서는 브라우저에서 신뢰하지 않음
- 공인인증기관(CA)이 발급·서명한 인증서만 브라우저에서 기본 신뢰
- CSR(인증서 서명 요청)을 CA에 제출 → 검증 후 서명된 인증서 발급
- 브라우저에는 신뢰할 수 있는 CA의 공개키가 내장되어 있음

### 내부망 환경
- 사내 CA를 운영해 내부 서비스 인증서 발급 가능
- 직원 브라우저에 사내 CA의 공개키를 설치해 신뢰 구성

### 클라이언트 인증서
- 서버가 클라이언트 신원 확인을 위해 요청할 수 있음(일반 웹서비스에는 드묾)
- 클라이언트도 CA 서명 인증서를 발급받아 서버에 제출
- 전체 인증서 발급·검증 인프라를 PKI(Public Key Infrastructure) 라고 함

### 파일 형식
- `.crt`, `.pem` → 공개키/인증서 파일
- `.key` → 개인키 파일
- 이름에 key가 없으면 주로 공개키·인증서, key가 있으면 개인키

## TLS in Kubernetes
### TLS 인증서의 구성 요소
- 3가지 주요 인증서
1. 서버 인증서 (Server CRT/PEM) – 서버 신원 검증
2. 루트 인증서 (CA CRT/PEM) – 모든 인증서 서명 및 신뢰 체계 제공
3. 클라이언트 인증서 (Client CRT/PEM) – 클라이언트 신원 검증
- 명명 규칙
- `.crt`, `.pem` → 공개키·인증서
- `.key` → 개인키 (파일명에 key 포함)

### 쿠버네티스 클러스터와 TLS 적용
- 목적: 마스터·워커 노드, 구성 요소 간 통신 전부 TLS로 암호화 및 상호 인증
- 필요성: 관리 툴(kubectl)과 API 서버 간, 구성 요소 간 보안 통신 보장

### 주요 서버 측 구성 요소 (서버 인증서 사용)
- kube-apiserver: 외부·내부 요청을 HTTPS로 처리 (apiserver.crt / apiserver.key)
- etcd: 클러스터 상태 저장, TLS 적용 (etcdserver.crt / etcdserver.key)
- kubelet: 워커 노드 API 엔드포인트, TLS 적용 (kubelet.crt / kubelet.key)

### 주요 클라이언트 측 구성 요소 (클라이언트 인증서 사용)
- 관리자(kubectl): admin.crt / admin.key
- scheduler: scheduler.crt / scheduler.key
- controller-manager: controller-manager.crt / controller-manager.key
- kube-proxy: kube-proxy.crt / kube-proxy.key
- 일부 서버도 다른 서버에 접근 시 클라이언트 인증서 사용 (예: kube-apiserver → etcd, kubelet)

### 인증서 발급 구조
- CA(Certificate Authority) 필요 → 모든 서버·클라이언트 인증서를 서명
- 보통 단일 클러스터 CA로 모든 인증서 서명
- CA 자체도 ca.crt / ca.key를 가짐

### 정리
- 서버 인증서: 서버가 자신을 증명
- 클라이언트 인증서: 클라이언트가 신원을 증명
- CA 인증서: 전체 신뢰 체계의 루트

## TLS in Kubernetes - Certificate Creation
### 인증서 생성 도구
- 사용 가능 도구: EasyRSA, OpenSSL, CFSSL
- 강의에서는 OpenSSL 사용

### CA(Certificate Authority) 인증서 생성
1. CA 개인키 생성 (`openssl genrsa -out ca.key …`)
2. CSR(인증서 서명 요청) 생성 – CN(Common Name)을 kubernetes-CA 등으로 지정
3. CA 자체 서명 인증서 생성 (`openssl x509 …`)
4. CA 키·인증서는 이후 모든 인증서 서명에 사용

### 클라이언트 인증서 생성 절차
- 대상
  - 관리자(admin) 사용자 (그룹: `system:masters`)
  - kube-scheduler (`system:kube-scheduler`)
  - kube-controller-manager (`system:kube-controller-manager`)
  - kube-proxy (`system:kube-proxy`)
- 과정
1. 개인키 생성
2. CSR 생성 (CN에 사용자명, OU에 그룹명 지정)
3. CA로 서명 → .crt 발급
- 목적: kubectl 등 API 서버 접근 시 사용자명·암호 대신 인증서로 인증
- 옵션 전달 또는 kubeconfig 파일에 구성

### 서버 인증서 생성
- etcd 서버
  - etcd-server.crt/key 생성
  - HA 환경일 경우 peer 인증서도 추가 생성
  - CA 루트 인증서 포함 필요
- kube-apiserver
  - 다양한 DNS·IP 별칭 포함 필요 → `openssl.cnf` 파일의 `subjectAltName` 섹션 설정
  - API 서버는 클러스터에서 모든 구성 요소와 통신하므로 서버 인증서와 클라이언트 인증서 모두 필요
- kubelet 서버
  - 각 노드별 kubelet.crt/key 생성 (이름: 노드명 기반)
  - kubelet이 API 서버 인증 시 사용할 클라이언트 인증서도 생성 (CN: `system:node:<노드명>`, 그룹: `system:nodes`)

### CA 루트 인증서 필요성
- 클러스터 내 모든 구성 요소는 상호 검증을 위해 CA 루트 인증서 사본을 가져야 함

### 생성된 인증서 활용
- kube-apiserver, etcd, kubelet 등 구성 요소 실행 시 관련 옵션(--tls-cert-file, --tls-private-key-file, --client-ca-file)에 지정
- 사용자·노드 권한 부여는 CSR 생성 시 CN/OU 필드로 결정
- 보통 kubeconfig에 통합 관리