# Install "Kubernetes the kubeadm way"
## Introduction to Deployment with 
- https://github.com/kodekloudhub/certified-kubernetes-administrator-course
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### kubeadm 개요
- 쿠버네티스 공식 부트스트랩 도구로, 모범 사례에 따라 클러스터 설치·구성을 자동화
- 인증서 발급, 구성 파일 생성, 컴포넌트 간 연결 설정 등 반복적·번거로운 작업을 처리

### 사전 준비
1. 호스트 프로비저닝
  - 물리 머신 또는 VM 형태로 마스터·워커 노드를 준비
2. 컨테이너 런타임 설치
  - 모든 노드에 Docker/CRI-O 등 컨테이너 런타임 설치
3. kubeadm 설치
  - 모든 노드에 kubeadm 바이너리 설치

### 마스터 초기화 (kubeadm init)
- 컨트롤 플레인 컴포넌트(API 서버, 컨트롤러·스케줄러 등)를 순서대로 설치·설정
- 클러스터 인증서와 `kubeconfig` 파일 생성

### 네트워킹 구성
- Pod 간 통신을 위한 CNI 플러그인 설치 (예: Calico, Flannel 등)
- 네트워크가 준비되어야 워커 노드가 정상 조인 가능

### 워커 노드 조인 (`kubeadm join`)
- 마스터에서 발급된 토큰·명령어를 이용해 각 워커 노드를 클러스터에 합류
- 인증·네트워크 설정이 자동 적용

### 이후 작업
- 클러스터가 완성되면 kubectl로 애플리케이션 배포 및 운영 시작
- 다음 단계로 로컬 데모를 통해 실제 설치·조인 과정을 확인