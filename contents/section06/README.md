# Cluster Maintenance
## OS Upgrades
클러스터 유지보수(소프트웨어 업그레이드·보안 패치 등)를 위해 노드를 안전하게 제거(또는 재부팅)하는 방법

### 노드 장애 시 기본 동작
1. 노드 다운 감지 후 최대 5분까지 대기
2. 5분 경과 시 해당 노드를 “죽은 것”으로 간주하고, 그 위의 Pod를 종료
3. ReplicaSet 소속 Pod는 다른 노드에 자동 재생성
4. 단독 Pod(ReplicaSet 미소속)는 그대로 사라짐

### 빠른 재부팅 전략
- 노드가 5분 이내에 복귀할 것이 확실할 때만 추천
- 다운 직후 재부팅 → 다시 온라인되면 기존 Pod가 그대로 돌아옴

### 안전한 유지보수 전략: 드레인(Drain)
1. Cordon: `kubectl cordon <노드명>`
  - 해당 노드를 “스케줄 금지” 상태로 표시하여 신규 Pod 배치 차단
2. Drain: `kubectl drain <노드명>`
  - cordon 수행 + 기존 Pod를 “정상 종료(퇴거)” → ReplicaSet Pod는 다른 노드로 재스케줄
3. 유지보수(재부팅·업그레이드) 수행
4. Uncordon: `kubectl uncordon <노드명>`
  - 노드를 “스케줄 가능” 상태로 되돌려, 이후 새 Pod가 배치될 수 있도록 허용

주의: Drain 후 자동으로 원래 노드로 돌아오지 않으니, 작업 완료 후 반드시 uncordon 실행

### 차단(Cordon)만 사용 시
- cordon만 하면 기존 Pod는 그대로 남고, 신규 Pod 배치만 막힘
- 긴 유지보수보다는 단기 차단 목적에 적합

## Cluster Upgrade Process
### 클러스터 업그레이드 개요
- 쿠버네티스는 컨트롤 플레인(참고: kube-apiserver, controller-manager, scheduler)과 노드 구성 요소(kubelet, kube-proxy)의 버전 스큐(skew)를 허용
  - kube-apiserver: 가장 높은 버전을 가져야 함
  - controller-manager, scheduler: apiserver보다 한 마이너 버전 낮게 허용
  - kubelet, kube-proxy: apiserver보다 두 마이너 버전 낮게 허용 (또는 같거나 한 버전 높게도 가능)
- 쿠버네티스는 최근 “세 개의 릴리스”만 지원
  - 예: 현재 1.12, 지원 범위는 1.10·1.11·1.12
- 업그레이드는 마이너 버전 단위로 순차 실행 권장
  - 1.10→1.11, 1.11→1.12 순으로 한 단계씩

### 업그레이드 시기
- 다음 마이너 릴리스가 나오기 전
- 지원 범위에서 벗어나기 전에
- 클라우드 매니지드(예: GKE)에서는 콘솔 클릭만으로도 가능

### 업그레이드 전략
1. 한 번에 전체 노드 업그레이드
  - 빠르지만 서비스 중단(Downtime) 발생
2. 롤링 업그레이드(노드별 순차 업그레이드)
  - kubectl drain → 패키지 업그레이드 → kubectl uncordon
  - 각 노드를 순서대로 업그레이드하며 무중단 목표
3. 블루-그린/그레이숄드 방식(새 노드 추가 후 교체)
  - 새 버전의 노드를 프로비저닝 → 기존 노드 드레인 및 제거

4. kubeadm을 이용한 업그레이드 절차
1. 준비
  - `kubeadm upgrade plan`으로 현재 클러스터 버전·지원 가능 버전 확인
2. kubeadm 자체 업그레이드
  - `apt-get install -y kubeadm=<목표버전>`
3. 컨트롤 플레인 업그레이드
  - `kubeadm upgrade apply <목표버전>`
  - API 서버·controller-manager·scheduler 자동 업그레이드
4. kubelet 업그레이드 (마스터 노드)
  - `apt-get install -y kubelet=<목표버전>` → 서비스 재시작
5. 워커 노드 업그레이드
  - 각 워커에 대해
    1. `kubectl drain <노드명> --ignore-daemonsets --delete-local-data`
    2. `apt-get install -y kubeadm=<목표버전>`
    3. `kubeadm upgrade node`
    4. `apt-get install -y kubelet=<목표버전>` → 서비스 재시작
    5. `kubectl uncordon <노드명>`

## Backup and Restore Methods
### 백업 대상
- 쿠버네티스 리소스 정의
  - Deployment, Pod, Service, ConfigMap, Secret, Namespace 등
  - 애플리케이션에 필요한 모든 객체의 manifest 파일
- 영구 스토리지 데이터
  - PVC로 연결된 볼륨 등 영구 저장소에 보관된 데이터

### 선언적(Manifest) 백업
1. 소스 코드 리포지토리 활용
  - 모든 리소스 정의 파일을 Git 등에 보관
  - 클러스터 손실 시 `kubectl apply -f` 만으로 재배치
2. 폴더 구조 관리
  - 앱별 디렉토리에 관련 객체 YAML 파일 집합
  - 협업·재사용 용이

### 명령적(API 쿼리) 백업
1. kubectl로 리소스 추출
  - `kubectl get all,cm,secret -n <namespace> -o yaml`
  - 스크립트화해 주기적으로 전체 네임스페이스의 리소스를 덤프
2. 자동화 도구 이용
  - Velero, Ark 등
    - API 서버 호출로 리소스 정의 + 볼륨 스냅샷까지 통합 백업 제공

### etcd 스냅샷 백업
1. etcd 데이터 디렉터리 지정
  - `--data-dir` 설정으로 스냅샷 대상 디렉터리 지정
2.	내장 스냅샷 커맨드
  - `etcdctl snapshot save <파일경로>`
  - `etcdctl snapshot status <파일>`로 백업 상태 확인

### 복구(Restore) 절차
1. API 서버 중단
2. 스냅샷 복원
  - `etcdctl snapshot restore <백업파일> \   --data-dir=<새데이터디렉터리> \   --initial-cluster=<새멤버구성>`
3. 클러스터 재구성
  - kube-apiserver, controller-manager, scheduler가 새 etcd 경로 사용하도록 설정
4. 서비스 재시작
  - kubelet 등 관련 데몬 재시작 후 정상 동작 확인
5. 인증서·엔드포인트 설정 유의
  - etcd TLS 인증서, API 서버 인증서·키 지정

### 선택 가이드
- 관리형 클러스터(GKE, EKS 등)
  - 사용자 등 접근이 제한적 → API 쿼리(velero) 방식 권장
- 자체 호스팅 클러스터
  - etcd 스냅샷 + manifest 백업 병행 → 완전 복구 가능