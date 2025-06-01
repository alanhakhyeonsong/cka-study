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