- [사전 배경 지식은 여기를 참고할 것.](Prerequisite.md)

# Networking
## Cluster Networking
- k8s 클러스터는 Master Node와 Worker Node로 구성.
- 각 노드는 최소 하나의 네트워크 인터페이스와 고유한 IP 주소가 필요.
- VM을 클로닝했을 경우 MAC 주소나 호스트 이름 충돌을 피해야 함.

### 열어야 할 주요 포트
- API 서버 : 6443
- Kubelet : 10250
- Control Plane → Kubelet : 10259
- Controller Manager : 10257
- Node Port 통신 (external) : 30000 ~ 32767
- etcd 서버 : 2379
  - 마스터 간 통신 : 추가로 2380 필요함.

참고로,
- 여러 마스터가 있다면 모든 관련 포트를 열어야 함.
- 공식 문서에서 포트 목록을 참고할 것.
- 문제가 발생하면 문서나 포트 설정을 먼저 점검.

### Docs
- [포트와 프로토콜 | Kubernetes](https://kubernetes.io/ko/docs/reference/networking/ports-and-protocols/)
- [클러스터 네트워킹 | Kubernetes](https://kubernetes.io/ko/docs/concepts/cluster-administration/networking/)
- [애드온 설치 | Kubernetes](https://kubernetes.io/ko/docs/concepts/cluster-administration/addons/)

## Pod Networking
- 클러스터 노드(Master, Worker) 네트워크는 이미 구성했다고 가정.
- Control Plane 구성요소(API 서버, etcd, kubelet 등)도 정상적으로 설정됨.
- 이제 애플리케이션(Pod)을 배포할 준비 단계

### 문제 상황
- Node 간 네트워크만 신경 썼지만, Pod 간 통신이 별도의 문제.
- 쿠버네티스는 Pod 네트워킹 규칙을 정의했지만 내장 솔루션은 없음.
- 사용자가 CNI 같은 네트워킹 솔루션을 구현/설정해야 함.

### Kubernetes Pod Network의 기본 요건
- **모든 Pod는 고유한 IP를 갖는다.**
- **Pod 간 통신은 IP를 통해 가능해야 한다.**
  - Pod가 어느 Node에 있든 상관없이 서로 IP로 접근.
- IP 대역/서브넷은 사용자 마음대로지만, 자동 할당 및 라우팅을 해결해야 함.

### 네트워크 구현 예시 (수동)
- Node 각각에 브리지 네트워크를 생성.
  - 각 브리지는 자체 서브넷 사용 (예: 10.240.x.x).
- 컨테이너(Pod)가 생성되면
  - Linux 네임스페이스 생성.
  - 브리지와 연결(가상 이더넷 페어 생성).
  - IP 주소 할당.
  - 기본 게이트웨이 경로 추가.
- 스크립트를 작성해 이 과정을 자동화 가능.
  - 컨테이너 생성 시 스크립트가 실행되어 네트워크 설정 수행.

### Node간 라우팅
- 각 노드의 Pod 서브넷을 다른 노드에서도 인식하도록 라우팅 테이블 구성.
- 단일 큰 네트워크 대역(예: 10.244.0.0/16)으로 설계.
- 작은 환경에선 서버 라우팅 테이블 수정으로 가능.
- 큰 환경에서는 라우터나 네트워크 장비가 경로를 관리.

### 자동화 : CNI 플러그인
- 수동 스크립트 방식으로는 수천 개의 Pod를 관리할 수 없음.
- CNI(Container Network Interface) 사용
  - 컨테이너 런타임이 컨테이너 생성 시 CNI 플러그인 호출.
  - 플러그인이 네임스페이스 생성, 브리지 연결, IP 할당, 경로 설정 자동 수행.
  - 생성 시 네트워크 추가, 제거 시 클린업 등 지원.

## CNI in Kubernetes
- 네트워크 네임스페이스, Docker 네트워킹 개념부터 시작해 표준의 필요성을 학습함.
- CNI는 컨테이너 네트워크 연결을 표준화한 규격.
- CNI 플러그인은 다양한 네트워크 솔루션을 지원.

### CNI의 역할
- 컨테이너 런타임(예: containerd, CRI-O)이 컨테이너 생성 시 CNI 플러그인을 호출.
- 컨테이너 런타임은 네임스페이스를 생성·연결하고 적절한 네트워크 플러그인을 실행.

### CNI 플러그인 구성
- 플러그인 바이너리: `/opt/cni/bin` 디렉터리에 위치.
  - 여기에는 bridge, DHCP, flannel 등 실행 가능한 플러그인들이 있음.
- 구성 파일: `/etc/cni/net.d` 디렉터리에 존재.
  - 어떤 플러그인을 어떤 설정으로 쓸지 정의.
  - 여러 파일이 있으면 알파벳 순서로 선택.

### 구성 파일 예시 (bridge 플러그인)
- 이름(name), 타입(type) → 'mynet', 'bridge' 등 지정.
- 필수 필드
  - IPAM 섹션
    - IP 관리 방식을 정의.
    - 예) 'host-local'은 로컬에서 관리, 'dhcp'는 외부 DHCP 서버 사용 가능.
  - Gateway 옵션
    - 브리지가 게이트웨이 역할 수행 여부.
  - NAT(IP masquerade)
    - 외부 접근 시 NAT 규칙 필요 여부.

컨테이너 런타임이 컨테이너 생성 시 CNI 플러그인을 호출해 네트워크를 자동 설정하며, 플러그인 바이너리와 구성 파일로 이를 관리한다.

## CNI Weave
- 단순한 네트워크에서는 Pod 간 통신이 라우터 경유로 가능.
- 하지만 Node와 Pod가 수백·수천 개면 관리가 비효율적.

### Weave의 비유적 설명
- 회사 사무실 간 소포 전달에 비유
  - 각 Node = 사무실
  - Node에 위브 요원(에이전트) 배치
  - 요원들은 서로 통신하며 다른 Node의 Pod IP·위치 정보 공유
  - 패킷 전달 시 캡슐화(encapsulation) → 목적지 Node로 전송 → 해제(decapsulation) → Pod 전달

### Weave의 동작 방식
- 각 Node에 Weave 에이전트(피어)가 설치되어 클러스터 토폴로지를 공유
- Pod간 통신
  - 위브가 패킷을 가로채서 캡슐화
  - 원격 노드의 위브 피어가 수신 및 복원해 대상 Pod로 전달
- Pod가 여러 브리지에 부착될 수도 있음

### Weave의 k8s 배포
- 일반 리눅스 서비스나 데몬으로도 배포 가능.
- 쿠버네티스에서는 **데몬셋(DaemonSet)** 으로 배포
  - 클러스터의 모든 노드에 Weave 피어를 자동 배포.
  - 단일 `kubectl` 명령으로 설치 가능.
  - 설치 후 로그 확인은 `kubectl logs` 활용.

## IP Address Management - Weave
- IP 주소 관리(IPAM)가 쿠버네티스(CNI) 네트워크에서 어떻게 작동하는지?
- Node의 가상 브리지 네트워크가 Pod(컨테이너)에 어떤 방식으로 IP 서브넷을 할당하는지?

### IPAM의 역할
- 컨테이너 네트워크 인터페이스(CNI)의 주요 역할 중 하나가 컨테이너(Pod)의 네트워크 네임스페이스에 IP를 할당하는 것.
- IPAM(주소 관리 플러그인)은 IP를 중복 없이 할당하고 추적.

### 관리 방식
- 쿠버네티스(CNI)는 IP 관리 방식을 강제하지 않고 플러그인 설계에 맡긴다.
- 가장 단순한 방법
  - 각 노드에 파일로 IP 목록을 저장
  - 스크립트로 읽고 IP 할당/회수 관리
- CNI 플러그인들은 이런 작업을 자동화해주는 IPAM 플러그인을 지원한다.

### 호스트-로컬 IPAM 플러그인
- 예제에서 다룬 기본 플러그인은 각 노드 로컬에서 IP 풀을 관리한다.
- 호스트 로컬 플러그인(host-local)은
  - 각 노드가 독립적으로 IP 풀을 유지
  - 파일 기반 저장
  - 스크립트나 CNI 구성에서 플러그인을 호출해 관리

### CNI 구성
- CNI 구성 파일에는 `ipam` 섹션이 포함.
  - 어떤 IPAM 플러그인을 쓸지
  - 어떤 서브넷을 할당할지
  - 라우트 정보 등을 명시
- 스크립트는 이 구성 파일을 읽고 알맞은 플러그인을 호출.
- 고정적으로 호스트-로컬을 쓰도록 하드코딩하지 않고 유연하게 플러그인을 교체 가능.

### Weave CNI의 사례
- Weave CNI 플러그인은 클러스터 전체에서 IP를 자동으로 할당하고 관리
- 예
  - 기본 IP 범위가 10.32.0.0/12
  - 이 큰 범위를 여러 노드가 나눠서 사용
  - 각 노드는 자신이 받은 서브넷 풀에서 Pod IP를 할당
- 분산된 여러 노드가 동등하게 IP 풀을 나눠 가짐
- 클러스터 수준에서 충돌 없이 관리

### 결론
- 쿠버네티스에서 Pod IP 관리는 CNI 플러그인이 전담
- IPAM 플러그인이 중복 방지, 서브넷 관리, 할당/회수 자동화
- 단일 노드에서는 호스트-로컬, 멀티 노드에서는 Weave 같은 분산형 플러그인 사용 가능
- CNI 구성 파일의 IPAM 섹션에서 원하는 플러그인을 선언하고, 스크립트나 쿠버네티스가 호출해 동작

## Service Networking
### Pod Networking
- 각 노드에서 네임스페이스와 가상 인터페이스 생성
- CNI가 서브넷 내에서 Pod에 IP 할당
- 오버레이 네트워크나 경로 설정으로 노드 간 Pod 통신 가능

### 서비스의 필요성
- Pod끼리 직접 IP로 통신하기엔 어렵다.
- 대신 서비스 객체를 만들어 Pod 그룹을 대표하도록 한다.
  - eg. 파란 Pod가 주황 Pod에 접근 → 주황 Pod가 속한 오렌지 서비스의 IP로 접속.

### 서비스 종류와 동작
- ClusterIP 서비스
  - 기본 서비스 타입.
  - 클러스터 내부에서만 접근 가능.
  - 서비스 IP 할당 → 클러스터 모든 Pod가 접근 가능.
  - 서비스는 특정 노드에 존재하지 않고 클러스터 전체에 논리적으로 존재.
- NodePort 서비스
  - 외부 트래픽을 클러스터로 노출.
  - 클러스터의 각 노드에 특정 포트를 열어 외부에서 접근 가능.
  - NodePort → ClusterIP → Pod로 트래픽 전달.

### 서비스는 실제 리소스가 아니다
- Pod처럼 네트워크 네임스페이스나 인터페이스를 가지지 않음.
- Service IP는 가상의 개념.
- 트래픽을 Service IP로 보내면 백엔드 Pod IP로 전달되는 규칙이 존재.

### IP 할당과 kube-proxy 역할
- 서비스가 생성되면
  - 쿠버네티스가 클러스터 IP 범위에서 IP를 할당.
  - 클러스터IP 범위는 Kube API 서버에서 지정 (`--service-cluster-ip-range`).
- 각 노드의 kube-proxy가
  - 서비스 생성/삭제 감지.
  - 해당 서비스 IP로 오는 트래픽을 백엔드 Pod IP로 전달하도록 규칙 생성.

### kube-proxy의 프록시 모드
- kube-proxy가 트래픽 전달을 구현하는 방식
  - userspace (구식)
  - IPVS (성능 좋음)
  - iptables (기본값, 널리 사용)
- iptables 모드 예시
  - 서비스 IP로 오는 트래픽 → DNAT 규칙으로 Pod IP와 포트로 변환.
  - 규칙은 각 노드의 iptables에 설치.

### 실제 예
- DB Pod가 IP 10.244.x.y를 가짐.
- 클러스터IP 서비스 생성 → 서비스 IP 10.103.x.y 할당.
- iptables 규칙이 추가
  - 목적지 서비스 IP → Pod IP로 NAT 변환.
  - 포트 매핑도 지원.
- NodePort 서비스도 유사하게
  - 모든 노드의 특정 포트 개방.
  - 해당 포트로 오는 트래픽을 서비스 → Pod로 전달.

### 정리
- Service는 클러스터 전역 개념.
- IP 할당 범위는 Kube API 서버 옵션으로 설정.
- kube-proxy가 각 노드에서 규칙 관리
  - 서비스 IP → Pod IP로 변환.
  - iptables / IPVS / userspace 방식 지원.
- Service와 Pod는 서로 다른 IP 범위를 가지며, IP 충돌 방지.

## DNS in Kubernetes
### DNS란 무엇인가?
-	도메인 이름을 IP 주소로 변환해주는 시스템.
-	`nslookup`, `dig` 같은 유틸리티 도구로 DNS 기록을 조회할 수 있음.
-	다양한 DNS 기록 유형이 존재.

### 쿠버네티스에서 DNS 구성
-	쿠버네티스는 클러스터 생성 시 기본 내장 DNS 서버(CoreDNS) 를 함께 배포.
-	Pod와 Service 간 통신에 DNS를 활용.

### 클러스터 내부 DNS 해상도
-	Pod → Service 혹은 Pod → Pod 간 IP 주소 없이 이름으로 통신 가능.
-	DNS 이름을 통해 클러스터 내 리소스를 식별하고 접근.

### 서비스에 대한 DNS 구성
-	Service가 생성되면 DNS 서버는 자동으로 도메인 이름을 생성.
-	네임스페이스 내부에서는 Service 이름만으로 접근 가능.
-	다른 네임스페이스에서 접근하려면 → `서비스이름.네임스페이스.svc.cluster.local`

### 네임스페이스와 도메인 구성
-	각 네임스페이스는 DNS의 하위 도메인으로 구성됨.
-	Service는 svc 하위 도메인에, Pod는 pod 하위 도메인에 포함.
-	클러스터 루트 도메인은 기본적으로 cluster.local.

### Pod에 대한 DNS
-	Pod는 기본적으로 DNS 기록이 생성되지 않지만 명시적으로 설정하면 가능.
-	Pod 이름 대신 IP를 대시(-)로 바꿔서 DNS 이름 구성 : `10-244-1-5.default.pod.cluster.local`

쿠버네티스 클러스터에서는 Pod와 Service가 DNS 이름을 통해 서로 통신할 수 있으며, 네임스페이스와 도메인 구조를 이해하면 이름 기반 접근 방식을 효율적으로 활용할 수 있다.

## CoreDNS in Kubernetes
### 왜 DNS가 필요한가?
-	Pod끼리 통신하려면 IP 주소를 알아야 하지만, Pod는 수시로 생성/삭제되기 때문에 정적인 `etc/hosts` 방식은 불가능.
-	따라서 중앙 DNS 서버(CoreDNS) 를 통해 Pod 및 Service 이름을 자동으로 해석하게 함.

### CoreDNS의 동작 원리
-	CoreDNS는 쿠버네티스 클러스터 내 `kube-system` 네임스페이스에 Pod로 배포됨.
-	보통 Deployment(또는 ReplicaSet) 형태로 2개 이상의 복제본이 실행됨.
-	설정 파일은 Corefile로, 다양한 플러그인(plugin) 을 통해 기능 확장 가능.

### Corefile 설정
-	CoreDNS는 Corefile을 통해 다음과 같은 기능을 설정
  -	오류 처리, 로깅, 모니터링
  -	kubernetes 플러그인을 통해 클러스터 내부 Service/Pod의 이름 해석
  -	클러스터 최상위 도메인: 기본값은 `cluster.local`

### Service 이름 해석 방식
-	Service 생성 시 자동으로 DNS 레코드가 생성됨.
-	네임스페이스 내에선 Service 이름만으로 접근 가능.
-	다른 네임스페이스에 있는 경우 FQDN(정규 도메인 이름) 필요: `서비스명.네임스페이스.svc.cluster.local`

### Pod 이름 해석 방식
-	Pod에 대한 DNS 기록은 기본값으로는 비활성화되어 있음.
-	설정 시 Pod IP를 대시(-)로 변환한 호스트명 형태로 생성 가능
- eg: `10-244-1-5.default.pod.cluster.local`

### DNS 서버 주소는 어떻게 설정되나?
-	각 Pod가 사용하는 DNS 서버는 CoreDNS Service의 클러스터 IP.
-	CoreDNS는 클러스터 내 kube-dns라는 이름의 클러스터IP Service로 노출됨.
-	Pod가 생성될 때 kubelet이 해당 DNS 주소로 `/etc/resolv.conf`를 자동 설정.

### 검색 도메인 (search domain)
- Pod 내부의 `/etc/resolv.conf`에 search 도메인이 자동 설정됨.
- eg: `default.svc.cluster.local`, `svc.cluster.local`, `cluster.local`
- 덕분에 `web-service`, `web-service.default`, `web-service.default.svc` 등 다양한 방식으로 접근 가능.

### 정리
-	쿠버네티스는 CoreDNS를 통해 Pod와 서비스 이름을 IP로 자동 변환할 수 있게 하고,
-	각 Pod는 자동으로 CoreDNS를 DNS 서버로 설정해, 손쉽게 다른 Pod나 서비스와 통신 가능.
-	설정 파일(Corefile)은 ConfigMap으로 관리되며, 필요시 수정 가능.

## Ingress
### 기초 시나리오: 간단한 NodePort / LoadBalancer 설정
-	애플리케이션과 MySQL 데이터베이스를 Kubernetes에 배포하고, 내부 통신은 ClusterIP 서비스(mysql-service)로 처리.
-	외부에서 접속하려면 NodePort 서비스를 이용해 포트(예: 38080)로 노출.
-	퍼블릭 클라우드(GCP 등)에서는 LoadBalancer 타입을 통해 외부 IP 자동 할당 가능.
-	그러나 **서비스가 많아질수록 로드밸런서 비용 증가, 복잡한 포트 관리, SSL 설정의 어려움 등 문제가 발생.**

### 이 문제들을 해결하기 위한 솔루션 → Ingress
-	Ingress는
  -	클러스터 외부에서 단일 URL 또는 도메인으로 접근 가능하게 하고,
  -	경로(path) 또는 호스트(host) 기준으로 트래픽을 내부 여러 서비스로 라우팅.
  -	SSL(HTTPS)도 중앙에서 쉽게 설정 가능.
-	일종의 L7(애플리케이션 계층) 로드밸런서.

### Ingress 구조
-	Ingress Controller: Nginx, Traefik, HAProxy 등. 클러스터에 배포해야 함 (기본 포함 X).
-	Ingress Resource: 트래픽 라우팅 규칙을 담은 Kubernetes 리소스 정의 (YAML).
-	컨트롤러는 Ingress 리소스를 감지하고 내부적으로 Nginx 설정을 자동 생성 및 반영.

### Nginx Ingress Controller 구성
-	Nginx Ingress Controller는 별도의 Deployment로 설치
  -	적절한 이미지, 실행 명령어, ConfigMap(옵션 설정), NodePort 서비스 등 포함.
  -	컨트롤러가 클러스터 내부의 Ingress 리소스를 감시하고, 요청을 적절한 서비스로 라우팅.

```yaml
apiVersion: extentions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
        - name: nginx-ingres-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
      args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
# ...
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443
    protocol: TCP
    name: https
  selector:
    name: nginx-ingress
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
# ...
```

<img width="1136" height="549" alt="Image" src="https://github.com/user-attachments/assets/6e7b2055-c0c2-4632-b85e-e4be30331f09" />

### Ingress Resource 예시
-	`Ingress` 정의 파일 (wear-ingress.yaml 등) 구성
-	`apiVersion`, `kind: Ingress`, `metadata`, `spec` 포함.
-	`spec.rules`: 특정 도메인(host)에 대해 경로(path)별로 어떤 서비스로 라우팅할지 명시.
-	`defaultBackend`: 규칙과 일치하지 않을 때 보내는 기본 서비스 (예: 404 Not Found 페이지용).

```yaml
# ingress-wear.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  backend:
    serviceName: wear-service
    servicePort: 80
---
# ingress-wear-watch.yaml (path 기반)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  rules:
  - http:
      paths:
      - path: /wear
        pathType: prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
      - path: /watch
        pathType: prefix
        backend:
          service:
            name: watch-service
            port:
              number: 80
```

```yaml
# (host 기반)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  rules:
  - host: wear-my-online-store.com
    http:
      paths: /
      pathType: prefix
      backend:
        service:
          name: wear-service
          port: 
            number: 80
  - host: watch-my-online-store.com
    http:
      paths: /
      pathType: prefix
      backend:
        service:
          name: watch-service
          port:
            number: 80
```

### 라우팅 방식 요약
-	Path 기반 라우팅
  -	하나의 도메인 내에서 `/wear`, `/watch` 등 URL에 따라 다른 서비스로 라우팅.
-	Host 기반 라우팅
  -	`wear.mystore.com`, `watch.mystore.com`처럼 도메인 이름에 따라 라우팅.
-	복합 구성도 가능: host + path 조합으로 다양한 세부 라우팅.

### 운영상의 이점
-	단일 Ingress Controller로 트래픽을 집중 관리 → 비용 절감
-	애플리케이션 코드 수정 없이 HTTPS 지원 및 경로 기반 라우팅
-	모든 설정을 YAML 정의 파일로 관리 가능 → GitOps, IaC 실현 가능

### 정리
Kubernetes Ingress는 외부 요청을 하나의 진입점으로 받아 경로(path) 또는 도메인(host)에 따라 내부 여러 서비스로 분산시키고, SSL 등 보안 기능까지 통합 제공하는 강력한 L7 로드밸런서 솔루션이다.

## Ingress - Annotations, Rewrite-Target
### 애플리케이션 설명
- watch 앱은 `<watch-service>:<port>/`에서 비디오 스트리밍 페이지를 제공.
-	wear 앱은 `<wear-service>:<port>/`에서 의류 페이지를 제공.

Ingress 설정을 다음과 같이 하고자 함.

|외부에서 접속한 주소 | 실제 내부 포워딩 대상|
|--|--|
|`http://<ingress-service>:<ingress-port>/watch`| `http://<watch-service>:<port>/`|
|`http://<ingress-service>:<ingress-port>/wear`|`http://<wear-service>:<port>/`|

여기서 중요한 점은, /watch, /wear 경로는 Ingress에서만 설정된 것이고, 실제 애플리케이션은 이 경로를 전혀 알지 못함.

### 문제 상황: rewrite-target 없이 라우팅할 경우
|외부 URL|내부 포워딩 결과|
|--|--|
|`/watch`|`/watch`|
|`/wear`|`/wear`|

즉, 내부 애플리케이션은 `/<경로>`가 붙은 URL을 받게 되고, 이 경로를 처리할 수 없으므로 404 Not Found 오류가 발생.

### 해결 방법: rewrite-target 사용
`rewrite-target` 옵션을 사용하면 Ingress에서 받은 경로를 내부로 전달할 때 원하는 경로로 바꿔줄 수 있다.

```yaml
nginx.ingress.kubernetes.io/rewrite-target: /
```

- 사용자가 `/watch`, `/wear`로 요청해도 실제로는 `/`로 바꿔서 내부 서비스에 전달.
- 이건 `replace("/path", "/")`와 같은 동작.
- 즉, Ingress의 path 설정을 `/pay`로 하면, 요청이 `http://.../pay`일 때 `http://pay-service/`로 전달되도록 경로를 수정하는 것.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /pay
        backend:
          serviceName: pay-service
          servicePort: 8282
```

`/pay`로 들어온 요청은 `pay-service:8282/`로 전달됨.

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /$2
...
path: /something(/|$)(.*)
```

- `/something/abc` → `/abc`
- `/something` → `/`

이 방식은 정규표현식을 활용하여 유연하게 경로를 처리

### 정리
|항목|설명|
|--|--|
|문제|Ingress로 경로를 나누어도, 내부 서비스가 해당 경로를 알지 못하면 404 오류 발생|
|해결|`nginx.ingress.kubernetes.io/rewrite-target` 어노테이션으로 **경로를 내부 서비스에 맞게 수정**|
|기본 예시|`/watch` → `/`, `/wear` → `/` 로 rewrite|
|정규표현식 예시|`/something/abc` → `/abc` 와 같은 유연한 매핑 가능|
|사용 목적|Ingress path를 기준으로 **다양한 서비스를 연결하면서도 내부 애플리케이션 수정 없이 트래픽 라우팅** 가능|

## Introduction to Gateway API
Ingress의 한계
-	멀티테넌시 지원 부족
-	단일 Ingress 리소스를 여러 팀이 함께 관리해야 해 충돌(조율) 발생
-	기능 확장성 제약
-	호스트 매칭·경로 매칭 외에 TCP/UDP 라우팅, 트래픽 분할, 헤더 조작, 인증, 속도 제한 등을 기본 지원하지 않음
-	컨트롤러별 어노테이션 의존성
-	NGINX, Traefik 등 컨트롤러 전용 어노테이션을 사용해야 해 쿠버네티스 스펙으로 검증·표준화 불가

### Gateway API 등장 배경
- Ingress의 한계를 극복하고, 레이어 4·7 라우팅과 멀티테넌시를 공식 Kubernetes 리소스로 제공하기 위해 탄생
- “차세대 쿠버네티스 인그레스·로드밸런싱·서비스 메시 API”를 지향

### 주요 리소스 모델
-	GatewayClass
	- 인프라 공급자(클러스터 관리자)가 정의
	- 어떤 컨트롤러(예: NGINX, Traefik, Envoy 등)로 트래픽을 처리할지 지정
-	Gateway
	- 클러스터 운영자가 인스턴스를 생성
	- 포트·프로토콜(HTTP, HTTPS, TCP 등)별 리스너 설정
-	Route (HTTPRoute 등)
	- 애플리케이션 개발자가 작성
	- Gateway가 수신한 트래픽을 어떤 서비스로, 어떤 조건(호스트·경로·가중치 등)으로 분기할지 정의
	- HTTPRoute 외에 TLSRoute, TCPRoute, UDPRoute, GRPCRoute 등 다양

### Gateway API의 특징 및 장점
- 선언적·구조화된 스펙
  - 어노테이션 불필요, 스키마 기반 검증 가능
- TLS 종료 및 리디렉션
  - Listener에 직접 TLS 설정 및 리디렉션 정책 포함
- 트래픽 분할(카나리아 배포)
  - 백엔드 서비스 버전별 가중치(예: 80% → v1, 20% → v2) 명시적 정의
- 필터 체인
  - CORS, 헤더 조작, 인증, 속도 제한 등 다양한 필터를 스펙 내에서 일관되게 구성

### 주요 구현체 현황
아래 항목들은 GA 상태 또는 구현 중
-	Amazon EKS, Azure Application Gateway, Contour, Easegress, Envoy Gateway, GKE, HAProxy Ingress Controller, Istio, Kong, Kuma, NGINX Gateway, Traefik 등