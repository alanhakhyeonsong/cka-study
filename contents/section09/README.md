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