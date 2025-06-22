## Core Concept
## Cluster Architecture
![Image](https://github.com/user-attachments/assets/16a69c26-cfee-42d0-ba03-a13b5447f72d)

- Master Node
  - `cloud-controller-manager`, `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`로 구성.
  - Control Plane 역할을 하는 Node에 해당하며, Manage, Plan, Schedule, Node 모니터링을 수행.
  - `kube-apiserver`를 통해 master node 내 컨트롤러들을 오케스트레이션.
- Worker Node
  - `kubelet`, `kube-proxy`, `kubernetes CRI`로 구성되고 동작 중인 Pod를 유지시키고 k8s 런타임 환경을 제공.
  - Control Plane에서 스케줄링되는 Pod들이 생성됨. 호스트 애플리케이션을 컨테이너 환경기반으로 실행시킨다는 표현이 맞을듯.
  - 

### ETCD
- 고가용 Key-Value Store.
- 각 컨테이너가 어떤 선박에(Node) 몇시에 적재되었는지 저장.

### kube-apiserver
- kubernetes api를 호출하는 k8s control plane 컴포넌트.

### kube-scheduler
- 노드가 배정되지 않은 새로 생성된 파드를 감지하고, 실행할 노드를 선택하는 control plane 컴포넌트.
- 스케줄링 결정을 위해서 고려되는 요소는 리소스에 대한 개별 및 총체적 요구 사항, 하드웨어/소프트웨어/정책적 제약, 어피니티(affinity) 및 안티-어피니티(anti-affinity) 명세, 데이터 지역성, 워크로드-간 간섭, 데드라인을 포함한다.

### kube-controller-manager
- 컨트롤러 프로세스를 실행하는 control plane 컴포넌트.
- 노드 컨트롤러: 노드가 다운되었을 때 통지와 대응에 관한 책임을 가진다.
- 잡 컨트롤러: 일회성 작업을 나타내는 잡 오브젝트를 감시한 다음, 해당 작업을 완료할 때까지 동작하는 파드를 생성한다.
- 엔드포인트슬라이스 컨트롤러: (서비스와 파드 사이의 연결고리를 제공하기 위해) 엔드포인트슬라이스(EndpointSlice) 오브젝트를 채운다
- 서비스어카운트 컨트롤러: 새로운 네임스페이스에 대한 기본 서비스어카운트(ServiceAccount)를 생성한다.

### kubelet
- 클러스터의 각 노드에서 실행되는 에이전트.
- pod에서 컨테이너가 확실하게 동작하도록 관리한다.
- Kubelet은 다양한 메커니즘을 통해 제공된 파드 스펙(PodSpec)의 집합을 받아서 컨테이너가 해당 파드 스펙에 따라 건강하게 동작하는 것을 확실히 한다. Kubelet은 쿠버네티스를 통해 생성되지 않는 컨테이너는 관리하지 않는다.

### kube-proxy
- 각 노드에서 실행되는 네트워크 프록시로, k8s의 서비스 개념의 구현부.
- 노드의 네트워크 규칙을 유지 관리한다. 이 네트워크 규칙이 내부 네트워크 세션이나 클러스터 바깥에서 파드로 네트워크 통신을 할 수 있도록 해준다.
- kube-proxy는 운영 체제에 가용한 패킷 필터링 계층이 있는 경우, 이를 사용한다. 그렇지 않으면, kube-proxy는 트래픽 자체를 포워드(forward)한다.

## ETCD
- Key Value Store.
- 데이터가 복잡해지면 json, yaml 등의 포맷에서 트랜잭션 처리.
- 2379 port

```bash
./etcd

./etcdctl set key1 value1
./etcdctl get key1
```

- k8s에선 클러스터에 대한 정보를 기록한다.
  - Nodes, Pods, Configs, Secrets, Accounts, Roles, Bindings, other
- 노드 추가, 파드 배포 등의 작업은 ETCD 서버에 업데이트 되어야 완료로 간주됨.
- 고가용성 환경에서 마스터 노드가 여러 개가 되고, ETCD도 여러 개가 되었을 때, 각각의 ETCD가 서로를 알 수 있도록 동기화해야 함.
- `kubectl get pods -n kube-system`

<img width="780" alt="Image" src="https://github.com/user-attachments/assets/305ff8a1-5bd3-4647-8253-48136d146dd4" />

- `kubectl exec etcd-master -n kube-system etcdctl get / --prefix -keys-only`

<img width="790" alt="Image" src="https://github.com/user-attachments/assets/33ecff2a-43fe-4e80-8a70-5b753374d89b" />

## Kube API Server
<img width="890" alt="Image" src="https://github.com/user-attachments/assets/6c9e1a2c-f753-4fcd-97fc-62f79161d43a" />

- etcd 저장소와 직접 상호작용하는 유일한 컴포넌트.

### 파드 생성 프로세스
1. kubectl 또는 API 요청
- 사용자가 kubectl 명령어나 API를 통해 파드 생성 요청
- 요청은 Kube API Server로 전달됨
2. API Server 처리
- 인증 및 권한 검증 수행
- 요청을 검증하고 ETCD에 저장
3. 스케줄러 작업
- kube-scheduler가 새로운 파드를 감지
- 적절한 Worker Node 선택 (리소스, 제약조건 등 고려)
4. kubelet 실행
- 선택된 노드의 kubelet이 파드 스펙 수신
- Container Runtime(containerd/docker)에게 컨테이너 생성 지시
5. 상태 업데이트
- kubelet이 파드 상태를 API Server에 보고
- API Server가 ETCD에 최신 상태 저장

## Kube Controller Manager
- Node Controller: 노드의 상태를 모니터링하고 관리
- Replication Controller: 지정된 수의 파드 Replication 관리
- Endpoints Controller: 서비스와 파드를 연결
- Service Account & Token Controllers: 새로운 네임스페이스에 대한 기본 계정과 API 접근 토큰을 생성

### Node Monitor Period, Node Monitor Grace Period
- Node Monitor Period: 노드 컨트롤러가 노드의 상태를 확인하는 주기 (기본값: 5초)
- Node Monitor Grace Period: 노드를 'Unreachable' 상태로 표시하기 전까지 기다리는 시간 (기본값: 40초)

### Pod Eviction Timeout
- 노드가 'Unreachable' 상태가 된 후, 해당 노드의 파드들을 제거하기까지 기다리는 시간 (기본값: 5분)
- 이 시간이 지나면 노드의 파드들이 강제로 종료되고 다른 정상 노드로 재스케줄링됨
- (Kubeadm) Controller Manager 는 마스터 노드의 kube-system 네임스페이스에 POD로 위치
- POD 정의 파일에서 옵션을 볼 수 있음 `/etc/kubernetes/manifests/kube-controller-manager.yml`
  - Non-Kubeadm : `/etc/systemd/system/kube-controller-manager.service`

### Kubeadm
- 자동화된 클러스터 설정 도구
- 쿠버네티스 컴포넌트들이 Pod로 실행됨
- 설정 파일: `/etc/kubernetes/manifests/` 디렉토리에 위치
- 컴포넌트 관리가 쿠버네티스 자체적으로 이루어짐
- 업그레이드와 관리가 상대적으로 쉬움

## Kube Scheduler
- 쿠버네티스에서 스케줄러는 Pod를 어느 노드에 배치할지 결정하는 컴포넌트.
- 직접 Pod를 배치하지 않고, 어떤 노드에 들어갈지 "선택"만 함.
- 실제 배치는 kubelet이 담당.

### 스케줄링이 왜 필요한가?
- 많은 Pod와 노드가 있는 환경에서, 적절한 리소스를 가진 노드에 적절한 Pod를 배치하는 것이 중요.
- Pod의 요구 사항은 다양하며, 노드의 특성(리소스, 목적지 등)도 다르다.

### 스케줄링 절차
1단계 - 필터링
- Pod가 배치될 수 없는 노드를 먼저 제외.
- 예: CPU/메모리 부족한 노드는 탈락.

2단계 - 점수 매기기 (우선순위 결정)
- 남은 노드에 대해 점수를 매겨 가장 적합한 노드를 선택합니다.
- 예: Pod를 배치한 후 남는 리소스가 많은 노드일수록 점수가 높음.

### 고급 기능
스케줄링 로직은 사용자 정의 가능하며, 리소스 요구사항, 제약 조건, 선호도 등 다양한 조건을 설정할 수 있다.

### 설치 및 구성
- Kube Scheduler 설치 방법
  - 쿠버네티스 배포판에서 `kube-scheduler` 바이너리를 다운로드해 실행.
  - 실행 시 스케줄러 구성 파일을 함께 지정.
- kubeadm으로 설치한 경우
  - 마스터 노드의 `kube-system` 네임스페이스에 Pod 형태로 배포됨.
  - 설정 파일 위치는 `/etc/kubernetes/manifests/` 디렉터리.
- 실행 중인 스케줄러의 설정은 프로세스 목록에서 `kube-scheduler` 프로세스를 검색해 확인할 수 있다.

## Kubelet
- Kubelet은 Node 내 컨테이너 운영을 총괄하는 핵심 에이전트.
- 비유하자면 노드라는 배의 선장 역할을 함.
  - 클러스터의 명령을 받아 컨테이너를 적재하거나 내리는 작업을 수행하고,
  - Pod와 컨테이너 상태를 모니터링하며 주기적으로 API 서버에 보고.
  - 노드와 마스터 간의 통신 창구이자 중재자.

### 주요 기능 및 역할
- Node 등록: 클러스터에 자신을 Node로 등록.
- 명령 수신: 스케줄러가 Pod 배치를 지시하면,
  - 컨테이너 런타임(Docker 등)을 호출하여,
  - 이미지를 pull하고 컨테이너를 실행.
- 상태 보고: Pod/Container 상태를 감시하고 kube-apiserver에 지속적으로 보고.

### 설치 및 실행
- kubeadm을 이용해 클러스터를 구성할 경우,
  - kubelet은 자동으로 설치되지 않음 → Worker Node에 수동 설치 필요.
- 설치 후, 서비스로 실행해야 하며,
  - Node에서 프로세스를 확인하고 kubelet 프로세스 및 실행 옵션을 점검할 수 있다.

### 보안 및 구성
kubelet 구성 및 인증서 생성, TLS 부트스트랩 과정.

## Kube Proxy
- kube-proxy는 각 노드에서 실행되는 네트워크 프록시 서비스.
- Pod가 아닌 가상의 서비스 객체에 도달할 수 있도록 네트워크 트래픽을 실제 백엔드 Pod로 라우팅하는 역할을 함.
- 쿠버네티스 클러스터의 서비스 트래픽 전달을 자동화한다.

### Pod Network와 Service
- 클러스터 내부 Pod 간 네트워크 통신은 Pod Network를 통해 이루어짐.
- Pod는 IP를 가지고 있지만, 이 IP는 영구적이지 않음 → 직접 접근은 위험.
- 따라서 **서비스(Service)**를 생성하여 Pod를 대표하게 하고, DNS 이름이나 서비스 IP를 통해 접근하게 한다.
- 단, 서비스는 가상 객체이며 실제 네트워크 인터페이스가 없음 → 바로 연결할 수 없음.

### kube-proxy의 역할
- 각 Node에서 실행되는 kube-proxy는 다음을 수행한다.
  - 새로운 Service가 생성되면 해당 트래픽을 백엔드 Pod로 전달하기 위한 `iptables` 규칙을 설정.
  - Service IP로 들어오는 트래픽을 Pod의 실제 IP로 포워딩.
- 예시: 10.96.0.12라는 Service IP가 10.32.0.15 Pod로 연결되도록 설정됨.

### 설치 및 배포
- 수동 설치: 쿠버네티스 배포 페이지에서 kube-proxy 바이너리를 다운로드하여 실행 가능.
- kubeadm 사용 시
  - kube-proxy는 DaemonSet 형태로 자동 배포됨.
  - 각 Node에 하나의 kube-proxy Pod가 항상 존재하도록 보장함.

DaemonSet 개념은 이후에 나옴.

## Pods with YAML
- 쿠버네티스는 YAML 파일을 통해 객체를 정의.
  - Pod, ReplicaSet, Deployment, Service 등 다양한 리소스를 생성 가능
- 모든 쿠버네티스 YAML 파일은 다음 4가지 최상위 필드를 포함해야 한다.
  - apiVersion: 사용할 쿠버네티스 API의 버전 (ex. v1, apps/v1beta1)
  - kind: 생성하려는 객체 유형 (ex. Pod, Service, Deployment)
  - metadata: 객체의 이름 및 레이블 등 부가 정보
  - spec: 객체의 상세 구성 정보

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: my-app
spec:
  containers:
    - name: nginx-container
      image: nginx

```

|Kind|Version|
|--|--|
|Pod|v1|
|Service||
|ReplicaSet|apps/v1|
|Deployment|apps/v1|

### metadata와 label 구성
- metadata는 객체 식별 정보를 담는다.
  - `name` : 리소스 이름
  - `labels` : 객체에 부여할 키-값 쌍의 레이블 목록
    - eg. `{ app: my-app }`
    - Pod를 그룹핑하거나 검색하는 데 유용 (ex. frontend, backend, db 등)
- 중첩 구조와 들여쓰기 중요
  - metadata 하위에 name, labels 등이 형제 관계로 잘 들여쓰기되어야 함.

### spec과 컨테이너 정의
- `spec`은 객체의 핵심 동작 정의를 포함.
  - Pod인 경우, 컨테이너 목록(containers)이 포함됨
  - `containers`는 배열이며, 각 컨테이너는 다음 정보를 가짐
    - `name`: 컨테이너 이름
    - `image`: 사용할 Docker 이미지 (예: nginx)

### 생성 및 확인 명령어
- Pod 생성

```bash
kubectl create -f pod-definition.yaml
```

- Pod 목록 확인

```bash
kubectl get pods
```

- Pod 상세 정보 확인

```bash
kubectl describe pod my-app-pod
```

→ 생성 시각, 적용된 라벨, 컨테이너 이미지, 이벤트 기록 등 확인 가능

### 자주 써먹을 명령어
```bash
# 해당하는 네임스페이스 내 모든 파드의 상세 목록 조회
kubectl get po -o wide

# 상세 출력
kubectl describe po my-pod

# nginx 파드에 대한 spec을 생성하고, pod.yaml이라는 파일에 해당 내용을 기록
kubectl run image --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl apply -f pod.yaml

# 네임스페이스 내에서 Redis라는 이름으로 실행중인 Pod 조회
kubectl get po --field-selector=metadata.name=Redis

# 'foo'라는 레플리카셋을 3으로 스케일
kubectl scale --replicas=3 rs/foo

# name=myLabel 라벨을 가진 파드와 서비스 삭제
kubectl delete pods,services -l name=myLabel

# 특정 리비전으로 롤백
kubectl rollout undo deployment/frontend --to-revision=2
```

## ReplicaSet
### 컨트롤러란?
- 쿠버네티스의 두뇌 역할
- 클러스터 상태를 모니터링하고 원하는 상태로 자동 조정하는 역할

### Replication Controller
- Pod의 원하는 수를 유지
- 고가용성 제공 (하나의 Pod가 죽어도 자동 복구)
- 로드 분산 목적의 Pod 복제 가능
- YAML 정의 파일 구성: apiVersion, kind, metadata, spec
- `spec.replicas` : 원하는 Pod 수
- `spec.template` : 복제에 사용될 Pod 템플릿
- `kubectl create -f 파일명.yaml` 명령어로 생성
- `kubectl get rc`, `kubectl get pods`로 상태 확인 가능

## ReplicaSet
- Replication Controller의 최신 대체 기술
- `apiVersion`: `apps/v1` 사용
- 핵심 구조는 유사하지만 **selector**가 필수
- `matchLabels`로 어떤 Pod를 모니터링할지 지정
- 이미 생성된 Pod도 동일 라벨이면 관리 가능
- template은 여전히 필요 (향후 Pod 복구용)
- 생성/삭제/확인 명령은 Replication Controller와 유사

### 스케일링 및 관리
- 수동 스케일링
  - YAML 파일 내 replicas 수정 후 `kubectl replace -f 파일명.yaml`
  - 또는 `kubectl scale rs/이름 --replicas=6`
- 자동 스케일링

### 정리
- 컨트롤러는 클러스터 상태를 자동 유지
- 복제 컨트롤러와 복제본 세트 모두 Pod를 복제해 고가용성을 보장
- 복제본 세트는 선택기 기능을 통해 더 유연하고 권장됨
- 템플릿은 향후 Pod 복구를 위한 정의로 항상 필요

## Deployment
- 기존에는 Pod 또는 ReplicaSet을 직접 사용해 애플리케이션을 배포했음.
- 하지만 제품 환경에선 더 정교한 기능이 필요함
  - 다수의 웹 서버 인스턴스 실행
  - 롤링 업데이트 (하나씩 점진적 업그레이드)
  - 롤백 기능 (업데이트 실패 시 이전 상태로 되돌리기)
  - 일시 정지 및 재개 (변경을 한꺼번에 적용하기 위함)

### Deployment란?
- 쿠버네티스의 고수준 Object
- ReplicaSet을 관리하고 자동으로 생성함
- 배포 시 자동으로 Pod를 생성함
- 주요 기능
  - 롤링 업데이트
  - 롤백
  - 일시 정지 및 재개

### Deployment 생성 방법
- 배포 정의 파일 작성 (YAML)
- 구성 요소
  - `apiVersion`: `apps/v1`
  - `kind`: `Deployment`
  - `metadata`: 이름, 레이블
  - `spec`: `replicas`, `selector`, `template`
- 배포 실행
  - `kubectl create -f [파일명].yaml`
  - `kubectl get deployments`로 생성 확인
- 자동 생성 확인
  - 배포 시 자동 생성된 ReplicaSet 확인
    - `kubectl get rs`
  - Pod 확인
    - `kubectl get pods`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

`kubectl create deployment httpd-frontend --image=httpd:2.4-alpine --replicas=3`

### Deployment, ReplicaSet, Pod 관계
Deployment → ReplicaSet → Pod

- Deployment는 전체 배포 전략을 관리하고,
- ReplicaSet은 실제 Pod의 복제 수를 관리함.

### 정리
- Deployment는 실무 배포를 위한 핵심 Object
- 롤링 업데이트와 롤백, 일시 정지 같은 실전 기능 제공
- 직접 ReplicaSet을 관리하지 않아도 됨
- `kubectl get all`로 생성된 모든 리소스 확인 가능

## Services
### 서비스란 무엇인가?
- Pod 간 통신과 외부 접근을 가능하게 하는 쿠버네티스 객체
- 애플리케이션 내부 구성 요소(프론트엔드, 백엔드, 데이터 소스) 간의 연결 제공
- 외부 사용자와 내부 시스템 간의 안정적인 연결을 유지

### 서비스의 주요 목적
- 내부 Pod들 간 통신 지원
- 외부 요청을 포드로 라우팅
- Pod의 수명주기(삭제/추가)에 따라 자동 갱신
- 내장 로드 밸런싱 기능 제공 (무작위 분산 방식)

### 서비스 유형
| 유형               | 설명                                 |
| ---------------- | ---------------------------------- |
| **ClusterIP**    | 클러스터 내부 통신 전용 (기본값)                |
| **NodePort**     | 외부에서 클러스터 노드의 IP + 지정 포트를 통해 접근 가능 |
| **LoadBalancer** | 클라우드 제공자가 지원하는 경우 외부 로드 밸런서 자동 생성  |

### NodePort 서비스 예제 흐름
- Pod 내부 웹 서버: TargetPort = 80
- 서비스 객체 내부 포트: Port = 80
- 노드 외부에 노출된 포트: NodePort = 30008 (기본 범위: 30000~32767)

→ 외부 사용자는 `http://노드IP:30008`로 접근

### 서비스 정의 파일 (YAML)
- `apiVersion`: v1
- `kind`: Service
- `metadata`: 서비스 이름 등
- `spec`
  - `type`: NodePort / ClusterIP / LoadBalancer
  - `ports`: port, targetPort, nodePort
  - `selector`: Pod 라벨과 일치시켜 연결

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```

### 라벨과 선택기
- 서비스는 selector로 라벨이 일치하는 Pod를 자동으로 찾아 연결
- Pod가 여러 개인 경우에도 동일 라벨을 기반으로 자동 라우팅

### 여러 노드에서 포드가 실행 중인 경우
- NodePort는 모든 노드의 동일 포트에 자동 설정됨
- 어떤 노드의 IP로 접근해도 요청이 적절한 Pod로 전달됨

### 정리
- 쿠버네티스 서비스는 Pod 간, 사용자와 애플리케이션 간 통신을 연결
- NodePort는 외부 접근을 가능하게 하며, 포트 매핑을 통해 요청 전달
- 서비스는 Pod 수나 위치 변화에도 유연하게 대응하며, 추가 구성 없이 자동 업데이트됨