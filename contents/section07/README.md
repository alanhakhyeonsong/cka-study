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

## View Certificate Details
### 인증서 확인 목적
- 새로 구성된 운영팀에서 인증서 관련 문제를 점검하기 위해 클러스터 전체 인증서 상태를 확인
- 만료, 발급자, 이름·대체이름, 조직 정보 등이 올바른지 확인 필요

### 환경 구성 이해
- 클러스터 배포 방식에 따라 인증서 관리 방식이 다름
  - 직접 설치한 경우: 관리자가 인증서 생성·배포
  - kubeadm 사용 시: 인증서와 구성 요소들이 Pod 형태로 자동 배포
- 점검 전, 어떤 도구와 배포 방식을 사용했는지 파악 필수

### 인증서 파일 위치와 정보 수집
1. kubeadm 환경
- `/etc/kubernetes/manifests` 폴더의 kube-apiserver 정의 파일에서 사용되는 인증서 파일 경로 확인
- 각 인증서 파일을 목록화하여 스프레드시트로 정리
2. 세부 정보 확인
- `openssl x509` 명령으로 인증서 디코딩
- 확인 항목
  - CN(이름) 및 Subject Alternative Name(SAN)
  - 유효 기간(만료일)
  - 발급자(CA)
  - 조직 정보
3. CA 발급자 확인
- kubeadm 기본 CA는 kubernetes-ca

### 점검 시 유의사항
- 모든 인증서에 대해 이름·SAN이 정확한지, 올바른 조직에 속하는지, 올바른 CA로 서명됐는지, 만료되지 않았는지 확인
- 세부 요구 사항은 쿠버네티스 공식 문서 참고

### 문제 발생 시 로그 확인
- 직접 설치한 경우: OS 서비스 로그 확인
- kubeadm 환경
  - `kubectl logs <pod>` 명령으로 구성 요소 Pod 로그 확인
  - 핵심 구성 요소(API 서버, etcd 등)가 다운돼 kubectl 사용 불가하면 Docker 명령(`docker ps -a, docker logs <container-id>`)로 확인

## Certificate API
### 기존 수동 방식
- 클러스터 관리자가 CA(루트 인증서 + 개인키)를 소유하고 직접 서명 처리
- 새 관리자가 접근 필요 시
  1. 새 관리자가 개인키 + CSR 생성
  2. 기존 관리자가 CA 서버에서 서명 후 인증서 반환
- 인증서 만료 시 동일 절차 반복
- CA 키·인증서는 안전한 서버에 보관해야 하며, 접근 권한이 있는 사람은 원하는 대로 사용자 생성 가능 → 보안 중요

### 자동화 필요성
- 사용자 수 증가·팀 규모 확장 시 수동 처리 비효율
- 인증서 갱신(회전)도 자동화 필요
- 쿠버네티스에는 이를 위한 내장 인증서 API가 있음

### 쿠버네티스 인증서 API 활용
- CertificateSigningRequest(CSR) 리소스 생성
  - 매니페스트 파일 작성 → `spec.request` 필드에 Base64 인코딩된 CSR 넣기
- CSR 생성 후 kubectl get csr로 요청 목록 확인
- 관리자가 `kubectl certificate approve <csr-name>`으로 승인
- 승인 후 CSR 객체에 서명된 인증서(Base64)가 포함됨 → 디코딩 후 사용자에게 전달
- 이로써 API 서버 접속 가능

### 인증서 API 처리 흐름
1. 사용자: 개인키·CSR 생성
2. 관리자: CSR 매니페스트 작성 및 쿠버네티스에 제출
3. 관리자는 CSR 승인
4. 서명된 인증서 Base64 → 디코딩 → 사용자 배포

### 인증서 API 백엔드 동작
- 컨트롤 플레인에서 **kube-controller-manager**가 담당
- 내부에 CSR-Approving, CSR-Signing 컨트롤러가 있음
- CSR 승인·서명에는 CA 루트 인증서와 개인키 필요

## KubeConfig
### kubeconfig 사용 이유
- 매번 kubectl 명령에 서버 주소, 클라이언트 인증서, 키, CA 인증서를 옵션으로 지정하는 것은 번거로움
- 이 정보를 `kubeconfig` 파일에 저장하면 명령에서 자동으로 참조 가능
- 기본 경로: 사용자 홈 디렉터리 `~/.kube/config`
- 기본 파일을 삭제하면 명시적으로 `--kubeconfig` 옵션으로 경로 지정 필요

### kubeconfig 파일 구조
- 형식: YAML, `apiVersion: v1`, `kind: Config`
- 세 가지 섹션
  1. clusters – 접근할 쿠버네티스 클러스터 정보 (서버 주소, CA 인증서 등)
  2. users – 해당 클러스터에 접근할 사용자 정보 (클라이언트 인증서/키 경로 등)
  3. contexts – 어떤 사용자가 어떤 클러스터에 접근할지 연결 정의 (옵션: namespace 지정 가능)
- 각 섹션은 배열 형태 → 여러 클러스터, 사용자, 컨텍스트를 한 파일에 정의 가능

### 컨텍스트 활용
- 컨텍스트 = 클러스터 + 사용자 + (옵션) 네임스페이스
- 현재 컨텍스트 설정: `kubectl config use-context <context-name>`
- 현재 컨텍스트는 kubeconfig의 `current-context` 필드에 저장
- `kubectl config view` 로 현재 구성 내용 확인 가능

### 인증서 지정 방법
- 일반적으로 파일 경로로 지정 (`certificate-authority`, `client-certificate`, `client-key`)
- 대안: 인증서 내용을 Base64 인코딩해 *-data 필드에 직접 포함 (`certificate-authority-data`, `client-certificate-data` 등)
- Base64로 인코딩된 인증서 데이터를 파일로 복원하려면 디코딩 수행

### 네임스페이스 설정
- 컨텍스트에 `namespace` 필드를 지정하면 해당 컨텍스트 사용 시 자동으로 특정 네임스페이스로 접속
- 여러 환경(개발/운영) 또는 여러 사용자 권한 전환 시 유용

## API Groups
### 쿠버네티스 API 개념
- Kube API Server는 클러스터와 상호작용하는 중심 컴포넌트
- `kubectl`이나 REST API를 통해 모든 작업이 API 서버로 전달됨
- 기본 포트: 6443
- eg: `/api/v1/pods` → Pod 목록 조회

### API 그룹 구조
- 핵심(Core) 그룹
  - 네임스페이스, Pod, 서비스, 이벤트, 노드, ConfigMap, Secret, PVC 등
- 명명된 그룹(Named Groups)
  - 기능별로 구분, 신규 기능도 여기에 포함
  - eg:
    - `apps` → Deployment, ReplicaSet, StatefulSet
    - `networking.k8s.io` → NetworkPolicy
    - `certificates.k8s.io` → CertificateSigningRequest
    - `authentication.k8s.io`, `authorization.k8s.io` 등
- 각 그룹 안에는 리소스(Resource) 들이 있으며, 리소스별로 수행 가능한 **동사(verbs)**가 있음
  - eg: `list`, `get`, `create`, `delete`, `update`, `watch` 등

<img width="1267" height="638" alt="Image" src="https://github.com/user-attachments/assets/7a6c4bc3-cb6c-48d3-b84e-0ac77083a1d5" />

### API 그룹 확인 방법
- API 서버 루트(`https://<API_SERVER>:6443/`)에 접근 시 지원 API 그룹 목록 확인 가능
- Named API Group 내부의 모든 리소스 목록도 반환 가능

### API 접근 방식
- 직접 curl로 접근 시 인증 메커니즘 필요 (인증서 파일 전달)
- 또는 kubectl proxy 사용
  - 로컬 포트 8001에서 프록시 서비스 실행
  - kubeconfig의 인증서와 자격 증명 사용
  - 명령줄에서 인증서 경로를 매번 지정할 필요 없음

### 용어 구분
- kube-proxy: 클러스터 내부에서 서비스와 파드 간 네트워크 트래픽을 처리하는 네트워크 프록시
- kubectl proxy: API 서버에 안전하게 접근하기 위해 로컬에서 HTTP 프록시를 띄워주는 명령

### 참고
- 쿠버네티스의 모든 리소스는 API 그룹·리소스·동사로 정의됨
- 이후 **권한 부여(Authorization)** 에서 이 구조를 기반으로 접근 허용/거부를 제어하게 됨

## Authorization
### 권한 부여의 필요성
- 인증(Authentication)으로 누가 접근할 수 있는지는 정해졌지만, 무엇을 할 수 있는지는 권한 부여로 결정됨
- 모든 사용자·서비스가 동일한 권한을 가져서는 안 됨
  - 예: 개발자는 앱 배포는 가능하지만 클러스터 네트워크/스토리지 설정 변경은 불가
  - 서비스 계정은 필요한 최소 권한만 부여

### 네임스페이스와 접근 제어
- 여러 팀/조직이 같은 클러스터를 공유할 경우 네임스페이스로 리소스를 분리
- 권한 부여 정책은 특정 네임스페이스에 대해서만 접근 허용/거부 가능

### 쿠버네티스 권한 부여 메커니즘
1. Node Authorizer
  - kubelet이 API 서버와 통신할 수 있도록 노드 요청 승인
  - 노드 이름은 `system:node:<노드명>`, 그룹은 `system:nodes`
2. ABAC(Attribute-Based Access Control)
  - 사용자/그룹과 권한을 직접 연결하는 정책 파일 작성
  - 정책 변경 시 파일 수정 후 API 서버 재시작 필요 → 관리 불편
3. RBAC(Role-Based Access Control)
  - 권한 집합을 역할(Role)로 정의하고 사용자/그룹을 역할에 매핑
  - 권한 변경 시 역할만 수정하면 일괄 반영 → 관리 효율적
4. Webhook Authorizer
  - 외부 시스템(예: Open Policy Agent)에 권한 판단 위임
  - API 서버가 사용자 정보와 요청 세부 정보를 외부로 전달, 결과에 따라 승인/거부
5. AlwaysAllow / AlwaysDeny
  - 모든 요청을 무조건 허용 또는 거부 (주로 테스트용)

ABAC 예시는 아래와 같음

<img width="1662" height="884" alt="Image" src="https://github.com/user-attachments/assets/c7b1ada4-5664-4406-b480-b2ebf8bd756d" />

RBAC 예시는 아래와 같음

<img width="1848" height="1018" alt="Image" src="https://github.com/user-attachments/assets/7ac89f51-34d9-44d7-9a84-243d83f88fbc" />

### 권한 부여 모드 설정
- API 서버 실행 시 `--authorization-mode` 옵션으로 설정
- 쉼표로 구분해 여러 모드 지정 가능
  - 예: `--authorization-mode=Node,RBAC,Webhook`
- 요청 처리 순서: 모드별로 순차 검사, 하나라도 승인하면 즉시 허용, 전부 거부 시 최종 거부

## Role Based Access Control (RBAC)
### RBAC 개념
- Role: 네임스페이스 단위에서 리소스에 대한 권한 집합을 정의
- RoleBinding: 특정 사용자/그룹을 Role과 연결해 권한을 부여
- 네임스페이스 범위에서만 권한이 적용됨 (클러스터 전역 권한은 ClusterRole/ClusterRoleBinding 사용)

### Role 생성
- API 버전: `rbac.authorization.k8s.io/v1`
- rules 항목: API 그룹, 리소스, 행동(verbs) 으로 구성
- 여러 규칙을 한 Role 안에 정의 가능
- 예: 개발자가 pods와 configmaps에 대해 `list`, `get`, `create`, `delete` 가능

### RoleBinding 생성
- 사용자(또는 그룹)를 Role에 연결
- `subjects`: 사용자 정보 지정
- `roleRef`: 연결할 Role 정보 지정
- Role과 RoleBinding은 동일 네임스페이스에서만 연동

### 권한 확인
- `kubectl auth can-i <verb> <resource>` 명령으로 확인 가능
- `--as <user>` 옵션으로 특정 사용자의 권한을 시뮬레이션 가능
- 리소스 이름 단위(resourceNames)로 세부 접근 제어 가능

### RBAC 예시

```yaml
# developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "delete"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["list", "get", "create", "delete"]
---
# developer-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: default
subjects:
  - kind: User
    name: dev-user        # 사용자 이름
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

권한 테스트 예시는 아래와 같음.

```bash
# 현재 계정이 pods를 생성할 수 있는지 확인
kubectl auth can-i create pods

# 특정 사용자(dev-user)가 configmaps를 조회할 수 있는지 확인
kubectl auth can-i get configmaps --as dev-user

# 특정 네임스페이스에서 테스트
kubectl auth can-i create pods --as dev-user --namespace test
```

## Cluster Roles
### Role과 RoleBinding
- 네임스페이스 범위에서 동작하며, 특정 네임스페이스 내 리소스 접근 권한을 부여.
- 네임스페이스를 지정하지 않으면 default 네임스페이스에 생성됨.
- 네임스페이스 범위 리소스 예: Pod, ReplicaSet, Deployment, Service, Secret 등.

### 클러스터 범위 리소스
- 네임스페이스에 속하지 않는 리소스.
- 예시: Node, PersistentVolume, 인증서 서명 요청(CSR), Namespace 자체 등.
- 생성 시 네임스페이스를 지정할 수 없음.

### ClusterRole과 ClusterRoleBinding
- 역할/역할 바인딩과 구조는 같지만, 클러스터 범위 리소스에 대한 권한을 다룸.
- 예시
  - 클러스터 관리자 역할: Node를 조회/생성/삭제.
  - 스토리지 관리자 역할: PV와 PVC 생성.
- 클러스터 역할 바인딩을 통해 사용자/그룹을 해당 ClusterRole과 연결.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]  # "" → core API 그룹
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-pods-global
subjects:
  - kind: User
    name: ramos               # 권한을 줄 사용자
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-reader           # 위에서 만든 ClusterRole 이름
  apiGroup: rbac.authorization.k8s.io
```

### 특징
- ClusterRole은 클러스터 범위뿐 아니라 네임스페이스 리소스에도 적용 가능.
  - 이렇게 하면 클러스터 전체 네임스페이스의 해당 리소스 접근 권한 부여 가능.
- 예시: ClusterRole로 “Pod 읽기” 권한 부여 시 → 모든 네임스페이스의 Pod 접근 가능.
- Kubernetes 설치 시 기본적으로 여러 ClusterRole이 생성됨.

### 명령어 참고
- `kubectl api-resources --namespaced=true` → 네임스페이스 리소스 목록.
- `kubectl api-resources --namespaced=false` → 클러스터 범위 리소스 목록.

## Service Accounts
### Service Account 개념
- 쿠버네티스 계정 종류
  - 사용자 계정(User Account): 사람이 사용(관리자, 개발자 등)
  - 서비스 계정(Service Account): 애플리케이션·자동화 도구 등 기계가 사용
- 예시: Prometheus, Jenkins, 사용자 정의 대시보드 앱 등이 K8s API 호출 시 사용.

### Service Account 동작 방식
- Service Account 생성 시 자동으로 **토큰(Secret 객체)** 이 함께 생성되어 API 인증에 사용됨.
- 쿠버네티스 내부에 배포된 애플리케이션은 토큰을 **Pod 내 볼륨(`/var/run/secrets/kubernetes.io/serviceaccount`)** 에 자동 마운트하여 사용.
- 기본 Service Account(default): 모든 네임스페이스에 존재하며, Pod 생성 시 자동 연결.
- 기본 Service Account은 제한된 권한만 가짐.
- Pod에 다른 Service Account을 사용하려면 serviceAccountName 필드로 지정.
- 기존 Pod의 Service Account은 변경 불가 → 재생성 필요.
(Deployment는 롤아웃으로 변경 가능)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-sa
  namespace: default
---
apiVersion: v1
kind: Pod
metadata:
  name: dashboard-pod
spec:
  serviceAccountName: dashboard-sa
  containers:
    - name: dashboard
      image: my-dashboard:latest
```

### Service Account 토큰 변경 사항 (K8s 1.22 / 1.24)
기존(1.22 이전)
- Service Account 생성 시 만료 없는 토큰을 가진 Secret이 자동 생성.
- Pod에 자동 마운트됨.
- 확장성·보안 문제
  - 유효 기간 없음 → 장기 노출 위험.
  - Service Account마다 Secret 개별 생성 → 관리 부담.

변경(1.22~)
- TokenRequest API 도입 (KEP-1205)
  - 토큰에 Audience 바인딩·유효 기간 포함.
  - Projection Volume을 통해 Pod에 마운트.
- Secret 자동 생성 대신 필요 시 명시적으로 토큰 요청.

변경(1.24~)
- Service Account 생성 시 자동으로 Secret/토큰 생성하지 않음.
- 토큰 필요 시
  - `kubectl create token <service-account>` 명령 사용.
  - 기본 유효기간: 1시간 (옵션으로 연장 가능).
- 여전히 만료 없는 토큰 Secret 생성 가능하나 권장하지 않음 (`kubernetes.io/service-account-token` 타입, `metadata.annotations`로 SA 연결).

### 보안 권장 사항
- 가능한 유효기간이 있는 토큰(TokenRequest API) 사용.
- 만료 없는 토큰은 API 사용 불가·특별 필요 시에만 생성.
- Service Account 토큰 Secret 생성은 최소화하여 보안 노출 방지.

## Image Security
### 이미지와 이름 규칙
- Kubernetes에서 Pod 배포 시 nginx 같은 이미지를 사용.
- 이미지 이름은 Docker의 네이밍 규칙을 따름.
  - 예: nginx → 실제로는 library/nginx.
  - library는 Docker Hub 공식 이미지 기본 계정.
  - 개인/회사 계정을 만들면 계정명/이미지명 형태 사용.

### 이미지 저장소(Registry)
- 이미지가 저장되는 곳은 레지스트리.
- 기본값: Docker Hub (docker.io).
- 그 외
  - GCR (gcr.io) – Google Container Registry
  - AWS ECR, Azure ACR, GCP Artifact Registry 등 클라우드 제공자별 레지스트리.
- 공개/비공개 모두 가능.
  - 사내 전용 앱은 프라이빗 레지스트리 사용 권장.

### 개인 레지스트리 접근
- 비공개 이미지를 사용하려면 로그인 필요.
- docker login 명령으로 사용자명/비밀번호 인증.
- Kubernetes에서 사용할 때는 자격 증명을 Secret으로 저장.

### Kubernetes에서 인증 처리
- Kubernetes는 Pod 실행 시 워커 노드의 Docker 런타임이 이미지를 가져옴.

```bash
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com

```
- 개인 레지스트리 접근 자격 증명을 **Secret 객체**로 생성.
  - 유형: docker-registry
  - 필드: 레지스트리 서버 주소, 사용자명, 비밀번호, 이메일.
  - 자주 쓰이는 Secret명으로 `regcred`가 있음.
  - 이렇게 하면 dockerconfigjson 형식으로 인코딩된 인증정보가 Secret 리소스로 저장됨.

```bash
kubectl get secret regcred --output=yaml
```
- 출력 시 `data[".dockerconfigjson"]` 항목에 base64 인코딩된 인증정보가 들어있음.
- Pod 정의 파일에 imagePullSecrets 항목으로 Secret을 참조.
- Pod가 실행될 때 Kubelet이 Secret을 이용해 레지스트리에서 이미지를 가져옴.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg-pod
spec:
  containers:
  - name: private-app
    image: myregistry.example.com/myuser/myapp:1.0
  imagePullSecrets:
  - name: regcred
```

- 매번 Pod마다 `imagePullSecrets`를 작성하기 번거롭다면 ServiceAccount에 설정해 네임스페이스 단위로 기본 적용 가능.

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

## Security Context
### 보안 컨텍스트 개념
- Docker 컨테이너 실행 시 보안 표준을 정의할 수 있음  
  → 예: 실행 사용자 ID, Linux capabilities(추가/제거 가능한 권한)
- Kubernetes에서도 동일하게 보안 컨텍스트(Security Context) 를 설정할 수 있음

### 적용 범위
- Pod 레벨
  - `spec.securityContext` 로 정의
  - Pod 안 모든 컨테이너에 적용됨
- Container 레벨
  - `spec.containers[].securityContext` 로 정의
  - 개별 컨테이너에 적용됨
- 우선순위
  - Pod + Container 둘 다 설정된 경우, Container 설정이 Pod 설정을 무효화함

### 예시
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```

- Pod 정의 파일에서 `securityContext` 필드 추가
  - Pod 수준: `runAsUser` 로 사용자 ID 지정
  - Container 수준: `capabilities` 옵션을 사용해 Linux 기능 추가/삭제

### 정리
- 보안 컨텍스트는 Pod/Container의 실행 환경 권한을 제어하는 기능
- 사용자 ID, 그룹 ID, Linux Capabilities 등 세부 실행 권한을 제한하거나 확장 가능
- 보안을 강화하고 최소 권한 실행(Least Privilege)을 실현하는데 중요한 요소

## Network Policy
### 네트워킹 기본 흐름
- 웹 앱 → API 서버 → DB 서버의 단순 아키텍처 예시.
- 기본 트래픽 종류
  - Ingress : 외부에서 들어오는 트래픽
  - Egress : 외부로 나가는 트래픽
- 기본적으로 Kubernetes는 모든 Pod 간 통신을 허용함.

### 보안 요구사항
- 예: 웹 서버가 DB 서버에 직접 접근하지 못하게 하고, 오직 API 서버만 DB 서버에 접근하도록 제한 필요.
- 이런 경우 NetworkPolicy 리소스를 사용.

### 네트워크 정책 개념
- 네임스페이스 안의 오브젝트로, Pod/ReplicaSet/Service와 비슷하게 동작.
- 특정 Pod에 레이블을 붙이고, NetworkPolicy에서 **PodSelector**로 해당 Pod 지정.
- 정책이 적용되면
  - 기본적으로 해당 Pod는 **모든 트래픽이 차단**됨.
  - 정책에 정의된 규칙과 일치하는 트래픽만 허용.

### 규칙 구성
- policyTypes: Ingress, Egress, 혹은 둘 다 지정.
- Ingress 규칙 예시 (DB Pod)
  - 특정 레이블(API Pod)에서 오는 트래픽만 허용.
  - 포트 3306 (MySQL)만 허용.
- 주의: Ingress만 정의하면 Egress는 막히지 않음.  
→ Egress 제한을 원한다면 반드시 `policyTypes`에 Egress 추가해야 함.

### 적용 예시
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api
    ports:
    - protocol: TCP
      port: 3306
```
- `role=db` 라벨이 붙은 Pod에 정책 적용.
- `role=api` Pod만 3306 포트로 접근 가능.

### 지원 여부
- 모든 CNI 플러그인이 NetworkPolicy를 지원하지는 않음.
- 지원하는 예: Calico, Cilium(강의에서 Calico/WeaveNet 등 언급됨)
- 지원하지 않는 경우(예: Flannel), 정책을 정의해도 실제로는 적용되지 않음.

## Custom Resource Definition (CRD)
### 리소스와 컨트롤러 개념
- 리소스(Resource): 예를 들어 Deployment 객체를 생성하면 etcd에 저장됨.
- 컨트롤러(Controller): 배포 컨트롤러가 etcd의 상태를 감시하고, 지정된 복제본 수만큼 파드를 생성/유지.
- 컨트롤러는 Kubernetes에 내장된 프로세스로 Go로 작성되어 있으며, 리소스의 원하는 상태를 실제 상태와 동기화함.

### 사용자 정의 리소스 (Custom Resource)
- Kubernetes 기본 리소스 외에도, 사용자가 임의의 객체(예: FlightTicket)를 정의할 수 있음.

```yaml
apiVersion: flights.com/v1
kind: FlightTicket
metadata:
  name: my-flight
spec:
  from: mumbai
  to: london
  tickets: 2
```
- 이렇게 만들면 etcd에 저장되지만, 실제 항공권 예약 API 호출은 일어나지 않음.

### 사용자 정의 컨트롤러 (Custom Controller)
- 특정 리소스 생성/삭제를 감시하고, 외부 API 호출이나 동작을 수행하는 코드.
- 예: FlightTicket 리소스가 생성되면 항공권 예약 API를 호출, 삭제되면 취소 API 호출.
- 기본 리소스의 Deployment Controller처럼 동작하지만, 사용자가 직접 작성해야 함.

### CRD (Custom Resource Definition)
- Kubernetes에 새로운 리소스 유형을 등록하는 방법.
- CRD 정의 예시
  - apiVersion: apiextensions.k8s.io/v1
  - kind: CustomResourceDefinition
  - metadata.name: flighttickets.flights.com
  - scope: Namespaced / Cluster
  - group: flights.com
  - names: kind(FlightTicket), singular(flightticket), plural(flighttickets), shortNames(ft)
  - versions: 여러 버전(v1alpha1, v1, …) 정의 가능, 스토리지 버전 지정 필요
  - schema: OpenAPI v3 스키마 기반으로 spec 필드의 구조/타입/검증 조건 정의 (예: from: string, to: string, tickets: integer)

### CRD의 역할
- CRD를 적용하면 `kubectl get flighttickets`, `kubectl create -f flightticket.yaml` 같은 명령 가능.
- 단, 이 상태에서는 etcd에 객체가 저장될 뿐 아무 동작도 하지 않음.
- 실제 동작을 위해서는 반드시 Custom Controller 가 필요.

### 정리
- CRD는 Kubernetes에 새로운 리소스 타입을 정의하고 유효성 검증 스키마까지 지정할 수 있는 도구.
- 하지만 CRD만으로는 단순히 etcd에 데이터만 저장될 뿐이며, 실제 동작을 위해서는 Custom Controller 가 필요.
- 즉, CRD = 새로운 리소스 정의, Custom Controller = 그 리소스에 반응하여 동작 수행.