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