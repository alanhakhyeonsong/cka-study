# Design and Install a Kubernetes Cluster
참고로 이 섹션은 시험에 필요한 사항은 아님. 참고로만 알아둘 것.

## Design a Kubernetes Cluster
1. 클러스터 설계 전 반드시 묻기
- 목적: 학습·개발·테스트용인지, 아니면 프로덕션(제품) 서비스용인지
- 호스팅 환경: 퍼블릭 클라우드(GCP/AWS/Azure) vs 셀프호스팅(On-premise)
- 워크로드 특성
  - 애플리케이션 수(소수 vs 다수)
  - 앱 종류(웹, 빅데이터, 분석 등)
  - 예상 네트워크 트래픽(지속적·폭발적)

2. 목적별 배포 권장
- 학습용
  - Minikube, 단일 노드 kubeadm, 또는 클라우드의 빠른 샌드박스 클러스터
- 개발·테스트용
  - 단일 마스터 + 복수 워커 노드
  - 직접 kubeadm 사용 혹은 GKE/AKS/EKS 같은 매니지드 서비스
- 프로덕션용 (고가용성)
  - 다중 마스터 노드 기반 HA 클러스터
  - 구성 도구: kubeadm, GKE, kOps(AWS), AKS(Azure) 등

3. 클러스터 규모 가이드라인
- 최대 지원 규모
  - 노드: 최대 5,000대
  - 파드: 최대 150,000개 (노드당 최대 100개)
  - 컨테이너 총합: 최대 300,000개
- 노드 리소스
  - 클라우드: 공급자가 노드 수에 맞춰 인스턴스 크기 자동 선택
  - 온프레미스: 동일한 사양 가이드라인 적용

4. 스토리지 설계 고려사항
- 고성능 워크로드: 로컬 SSD
- 동시 다중 액세스: 네트워크 기반 스토리지
- 공유 볼륨: 쿠버네티스 Persistent Volume & StorageClass 활용
- 스토리지 클래스 정의 후 애플리케이션별로 지정

5. 노드 유형 및 운영체제
- 물리 vs 가상 머신 (VirtualBox, 클라우드 VM 등)
- 운영체제: 64비트 Linux 필수
- 마스터 노드 역할
  - 기본적으로 컨트롤 플레인만 호스팅
  - 프로덕션에서는 Taint 설정해 워크로드 스케줄링 차단

6. 대규모·고가용성 확장
- 제어 평면 분리: 대형 클러스터의 경우 별도 물리/논리적 분리

7. 시험·실무 팁
- 클러스터 관련 수치는 문서에서 조회 가능하므로 암기 불필요
- 공식 문서 링크 참조 권장

## Choosing Kubernetes Infrastructure
1. 로컬/학습용 배포
- 직접 설치: Linux 바이너리 수동 설치
- Minikube: 단일 노드 VM 안에 쿠버네티스 구성 요소 실행
- kubeadm: 이미 프로비전된 VM(물리·가상)에 단일·다중 노드 클러스터 빠르게 구성
- Windows 환경: Hyper-V/VMware/VirtualBox 위 Linux VM 또는 Docker-in-Docker 방식

2. 턴키·매니지드 솔루션
- kOps (AWS), kubeadm on cloud VMs: VM 프로비전부터 클러스터 구성·업그레이드 자동화
- BOSH, OpenShift (on-prem), VMware PKS: HA 클러스터 설치·관리 툴
- Vagrant: 멀티-클라우드에 동일한 스크립트로 배포

3. 퍼블릭 클라우드 호스팅 서비스
- GKE (Google), EKS (AWS), AKS (Azure), OpenShift Online (Red Hat)
- 공급자가 VM 관리·패치·업그레이드 제공
- 클릭 몇 번으로 프로비저닝·스케일링·업그레이드 가능

4. 선택 기준
- 목적: 학습·테스트용 vs 개발용 vs 프로덕션용
- 운영 책임 범위: 직접 운영 vs 일부 위임(턴키) vs 완전 매니지드
- 조직·예산·보안 정책: 온-프레미스 제약 vs 퍼블릭 클라우드 활용

5. 실습 예시
- 학습용으로 LocalBox(VirtualBox) 기반 3노드 클러스터 구성
- 마스터 1, 워커 2 대를 노트북에 가상 머신으로 프로비전

## Configure High Availability
### 고가용성(HA) 필요성
- 단일 마스터 장애 시 문제점
- API 서버, 컨트롤러·스케줄러가 모두 다운 → ReplicaSet 재생성, 신규 스케줄링, kubectl 접근 불가
- 워커 노드 위에 이미 실행 중인 파드에는 영향 없으나, 관리·운영 기능이 마비됨
- 해결책: 컨트롤 플레인(마스터) 구성 요소 전반에 중복을 두어 단일 실패점(SPOF) 제거

### 컨트롤 플레인 중복 구성
1. API 서버
  - 여러 인스턴스를 Active–Active 모드로 실행
  - 요청 분산을 위해 마스터 앞단에 Load Balancer(Nginx, HAProxy 등) 필요
2. Controller-Manager & Scheduler
  - Active–Passive 모드로 실행
  - 리더 선출(Leader Election)
  - Kubernetes Lease API 사용 (--leader-elect=true)
  - 기본 임대 기간: 15s, 갱신 기한: 10s, 재시도 간격: 2s
  - 장애 시 대체 인스턴스가 자동으로 리더가 되어 제어 기능 지속

### etcd 토폴로지 옵션
1. Stacked Topology
  - 각 마스터 노드에 etcd 멤버가 함께 실행
  - 구성·관리 간단하나, 마스터 노드 전체 장애 시 etcd 가용성 저하
2. External Topology
  - etcd 클러스터를 별도의 서버 세트에 배치
  - 제어 평면 노드와 분리되어 더 높은 내결함성 제공
  - 구성 난이도와 운영 비용 상승

### HA 클러스터 설계 요소 정리
- 마스터 개수: 최소 3대 권장 (다수의 컨트롤 플레인 인스턴스)
- Load Balancer: API 서버 앞단에 배치하여 트래픽 분산
- Leader Election: Controller-Manager/Scheduler에 적용하여 중복 실행 시 과잉 동작 방지
- etcd 구성: 서비스 규모·운영 여건에 맞춰 Stacked vs External 선택

## ETCD in HA
### etcd 개요
- 분산 키-값 스토어로, JSON/YAML 형태의 문서를 저장
- 간단·안전·고속이며 여러 노드에 동일한 데이터 복제

### 데이터 복제와 일관성
- 읽기(Read): 모든 노드에서 동일한 데이터 즉시 조회 가능
- 쓰기(Write)
  1. 리더(Leader) 선출 후 쓰기 요청을 리더가 처리
  2. 리더가 팔로워(Follower)들에게 변경 사항 전파
  3. 과반수(Quorum) 노드에 복제 완료된 후 쓰기 최종 확정

### 리더 선출 (Raft 알고리즘)
- 각 노드에 무작위 타이머 설정 → 먼저 만료된 노드가 후보자로 나서 투표 요청
- 과반수 찬성 시 리더 임명
- 리더는 주기적 하트비트(Heartbeat) 송신 → 팔로워 안정화
- 네트워크 분할 혹은 리더 장애 시 재투표로 새 리더 선출

### 쿼럼(Quorum)과 내결함성
- 쿼럼 계산: ⌊N/2⌋+1 (예: 3노드→2, 5노드→3)
- 홀수 노드 권장
  - 짝수일 경우 특정 분할 상황에서 어느 쪽도 과반 달성 못해 전체 장애 발생
  - 3·5·7개 노드 구성 시 네트워크 분할에도 한쪽에 과반 유지
- 내결함성(Fault Tolerance)
  - N=3 → 1노드 고장 허용
  - N=5 → 2노드 고장 허용

### etcd 설치·구성
1. 바이너리 압축 해제 및 디렉터리 구조 생성
2. TLS 인증서 배치 (peer/API 서버 통신용)
3. 서비스 설정 (initial-cluster 옵션에 각 노드 피어 정보 지정)
4. etcdctl v3 사용:
  - `export ETCDCTL_API=3`
  - `etcdctl put <key> <value>` / `etcdctl get <key>` / `etcdctl get “” --prefix` 등

### 설계 권장
- 최소 3개 노드로 HA 환경 구성
- 홀수 노드 선택(3·5·7…)
- Stacked Topology: 마스터 노드에 etcd 멤버 함께 실행(간단하지만 마스터 전체 장애 시 클러스터 영향)
- 대규모·운영 환경에서는 External Topology(별도 etcd 전용 서버) 고려