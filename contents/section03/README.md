# Scheduling
## Manual Scheduling
### 스케줄러가 없는 경우
- 클러스터에 내장 스케줄러가 없으면 Pod는 Pending(보류) 상태로 남는다.
- 스케줄러가 하는 일
  - Node 이름 필드가 비어 있는 Pod를 찾아서
  - 스케줄링 알고리즘으로 적합한 Node를 선택
  - Binding 객체를 만들어 Node 이름을 Pod에 지정

### 수동 스케줄링
- Node 이름 필드 지정
  - `spec.nodeName` 필드에 Node 이름 지정
  - 단, Pod 생성 시에만 지정 가능함. 생성 후에는 수정 불가.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
      - containerPorts: 8080
  nodeName: node02
```

- 이미 생성된 Pod에 노드 할당
  - 생성 후에는 `spec.nodeName`을 직접 수정할 수 없음.
  - 대신 Binding 객체를 사용
    - 대상 노드를 명시
    - Kubernetes API에 바인딩 객체를 POST
    - 스케줄러가 하는 바인딩 과정을 수동으로 수행
  - 포맷
    - 바인딩 객체는 JSON 형식
    - YAML을 JSON으로 변환 후 API 요청

## Labels and Selectors
Label과 Selector는 쿠버네티스에서 오브젝트를 그룹으로 묶고 필터링하기 위한 표준적인 방법

- Label
  - 각 오브젝트(예: Pod)에 붙이는 key-value 형태의 속성.
  - 예: `app=app1`, `color=green`, `type=mammal`
  - 원하는 만큼 라벨을 추가할 수 있음.
  - 유튜브 태그나 온라인 상점의 상품 필터처럼 동작.
- Selector
  - 특정 라벨 조건을 만족하는 오브젝트만 선택/조회.
  - 예: `kubectl get pods -l app=app1`
  - 라벨로 필터링해 원하는 객체만 보거나 제어할 수 있음.

- 오브젝트가 수백~수천 개가 될 때 앱별, 기능별, 유형별로 그룹화/필터링 가능.
- ReplicaSet, Deployment, Service 등 쿠버네티스 오브젝트끼리 연결할 때 사용.
  - ReplicaSet이 관리할 Pod를 선택하려면 Pod의 라벨과 ReplicaSet의 selector가 일치해야 함.
  - Service도 Selector를 통해 특정 Pod 그룹을 찾고 라우팅.

### 정의 예시
- Pod YAML의 `metadata.labels`에 라벨 정의.
- ReplicaSet YAML
  - `metadata.labels`: ReplicaSet 자체 라벨
  - `spec.selector.matchLabels`: 관리할 Pod의 라벨 지정
  - `spec.template.metadata.labels`: 생성할 Pod의 라벨

주의 사항
- ReplicaSet의 selector와 template.metadata.labels가 반드시 일치해야 함.
- 필요하면 여러 라벨을 조합해 더 정밀하게 선택.

### 주석(Annotations)
- 라벨과 달리 필터링이 아니라 정보 기록 용도.
- 예: 버전, 빌드 정보, 연락처.

## Taints and Tolerations
**특정 노드에 어떤 Pod를 배치할지 제한하거나 허용하는 방법**

- Node = 사람
- Pod = 벌레
- Taint = 방충 스프레이 (Node에 설정)
  - Pod가 못 오도록 막음
  - 예: `kubectl taint nodes node1 app=blue:NoSchedule`
  - 효과(Effect)
    - NoSchedule: Pod 스케줄 거부
    - PreferNoSchedule: 스케줄러가 피하려고 시도
    - NoExecute: 새 Pod 스케줄 거부 + 기존 Pod 퇴거
- Toleration = 내성 (Pod에 설정)
  - 특정 Taint를 "견딜 수 있는" Pod만 배치 가능
  - Pod YAML에 `tolerations` 추가

```yaml
tolerations:
- key: "app"
  operator: "Equal"
  value: "blue"
  effect: "NoSchedule"
```

### 동작 방식 예시
- Node 1에 `app=blue:NoSchedule` 테인트 설정
  - 기본적으로 모든 Pod가 거부
- 특정 Pod D에 toleration 추가
  - D만 Node 1 배치 가능
- 나머지 Pod A, B, C는 거부되어 다른 Node로 이동

### 명령 예시
- Node에 테인트 추가
  - `kubectl taint nodes node1 app=blue:NoSchedule`
- Pod YAML에 톨러레이션 추가

```yaml
tolerations:
- key: "app"
  operator: "Equal"
  value: "blue"
  effect: "NoSchedule"
```

### 주의할 점
- 테인트와 톨러레이션은 "허용"의 개념만 있음
- 특정 노드에 강제 배치하지는 않음
- Pod가 테인트를 "견딜 수 있게" 해줄 뿐
- 특정 노드에 강제로 배치하려면 노드 셀렉터/노드 어피니티 사용

### 마스터 노드의 테인트
- 쿠버네티스 설치 시 마스터 노드에는 기본적으로 테인트가 걸림
  - 일반 워크로드가 마스터 노드에 배치되지 않도록 방지
  - 필요하면 이 테인트를 제거하거나 톨러레이션 추가 가능
  - 확인 : `kubectl describe node <master-node-name>`

테인트와 톨러레이션은 노드에 어떤 Pod를 못 들어오게 하거나 특정 Pod만 허용하기 위해 사용하는 쿠버네티스 스케줄링 제어 도구다.

## Node Selectors
**특정 Pod를 특정 노드에 제한해서 배치하는 가장 간단한 방법**

### 문제 상황
- 클러스터에 노드가 3대 있음
  - 2대 = 작은 노드 (적은 리소스)
  - 1대 = 큰 노드 (많은 리소스)
- 특정 Pod(C)가 큰 노드에서만 실행되길 원함
  - 이유: 리소스 많이 쓰는 데이터 처리 작업
- 기본 설정
  - 스케줄러는 Pod를 아무 노드에나 배치함
  - 원하는 제어가 안 됨

### 해결책 1. Node Selector
- Pod가 특정 노드에서만 실행되도록 제한
- Pod 정의 파일(spec)에 `nodeSelector` 추가

```yaml
spec:
  nodeSelector:
    size: large
```

- 전제 조건 : 노드에 라벨이 있어야 함
- 노드에 라벨 부여 : `kubectl label nodes <노드이름> size=large`
- 스케줄러는 이 라벨을 가진 노드만 선택

- 간단하고 직관적임
- 하지만 제약적임
  - 오직 하나의 key-value 쌍 조건만 가능
  - "큰 노드이거나 중간 노드" 같은 복잡한 조건은 불가능
  - "작지 않은 노드" 같은 부정 조건도 불가능
- Node Selector는 가장 기본적이고 간단한 노드 선택 방법
더 복잡한 스케줄링 조건이 필요하다면 **노드 어피니티(Node Affinity)**와 **안티 어피니티(Anti-affinity)**를 사용해야 한다.

## Node Affinity
**Pod를 특정 노드(또는 노드 집합)에 더 정밀하게 배치 제어하기 위한 고급 스케줄링 규칙**

- Node Selector는 단순
- 하나의 key=value 라벨만 매칭
- 고급 조건 불가능
  - "큰 노드나 중간 노드" → NodeSelector는 불가능
  - "작은 노드는 제외" → NodeSelector는 불가능

### Node Affinity
- Pod를 특정 노드에 배치할 더 복잡한 규칙을 지원
- NodeSelector와 유사하지만
  - 복잡한 표현식 지원 (연산자 사용)
  - `In`, `NotIn`, `Exists`, `DoesNotExist`
  - 여러 값 허용 (예: `size in [large, medium]`)

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: size
          operator: In
          values:
          - large
          - medium
```

### Node Affinity 유형
- `requiredDuringSchedulingIgnoredDuringExecution`
  - 필수 조건
  - 스케줄러가 반드시 이 조건을 만족하는 노드에만 Pod 배치
  - 조건 불일치 → 스케줄 실패 (Pod Pending 상태)
- `preferredDuringSchedulingIgnoredDuringExecution`
  - 선호 조건
  - 스케줄러가 최대한 조건 맞춰서 배치 시도
  - 조건 만족 노드 없으면 아무 노드에나 배치 가능
- 특징
  - 둘 다 스케줄 시에만 평가됨
  - 스케줄 이후 노드 라벨이 변경돼도 이미 배치된 Pod는 그대로 실행됨

### 예제
- Node Label 추가  
  → `kubectl label nodes <노드명> size=large`
- Pod 정의 예시

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - large
```

- Node Affinity는 NodeSelector보다 더 강력하고 유연함.
- 스케줄링 시 복잡한 조건 지원.
- 다만, 스케줄 이후에는 라벨 변화 무시(Pod는 계속 실행됨).
- 실행 단계에서 조건 검증하는 기능은 차후 정리.

## Resource Limits
### 스케줄링 개요
- 클러스터 환경: 3개의 노드(각각 가용 CPU/메모리 보유)
- 스케줄러 역할:
  - Pod가 요구(request)하는 리소스 양과
  - 각 노드의 사용 가능한 리소스를 비교
  - 충분한 곳에만 Pod를 배치
  - 부족하면 Pod 스케줄을 보류(“Pending”)

### 리소스 요청(Request)
- 컨테이너가 최소한으로 보장받아야 할 CPU/메모리
- 스케줄러는 이 값을 기준으로 적절한 노드를 선택

```yaml
resources:
  requests:
    cpu: "1"        # 1 vCPU
    memory: "1Gi"   # 1 Gibibyte
```

### 리소스 한도(Limit)
- 컨테이너가 최대 사용할 수 있는 CPU/메모리
- 초과 시 CPU는 쓰로틀링(throttling), 메모리는 OOM(kill) 발생

```yaml
resources:
  limits:
    cpu: "2"         # 최대 2 vCPU
    memory: "512Mi"  # 최대 512 Mebibyte
```

### 단위 정리
- CPU
- 1 CPU = 1 vCPU (AWS, GCP의 코어 혹은 하이퍼스레드 1개)
- m 단위: 1000m = 1 CPU, 최소 1m
- 메모리
- SI 단위: 1 G = 1 000 000 000 바이트
- Binary 단위: 1 Gi = 1 024 Mi (1 GiB = 1 073 741 824 바이트)

### 기본 동작(요청/한도 미설정 시)
- 디폴트: 요청(request), 한도(limit) 모두 없음
- Pod는 필요에 따라 노드의 모든 리소스 소비 가능
- 다른 워크로드 혹은 시스템 프로세스가 영향을 받을 수 있음

### LimitRange
- 네임스페이스 단위로 디폴트 요청/한도, 최소·최대값 지정

```yaml
kind: LimitRange
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "500m"
      memory: "500Mi"
    default:
      cpu: "1"
      memory: "1Gi"
    min:
      cpu: "100m"
      memory: "100Mi"
    max:
      cpu: "2"
      memory: "2Gi"
```
- 적용 이후 생성되는 새 Pod에만 반영

### ResourceQuota
- 네임스페이스 전체가 사용할 수 있는 총 요청(request), 총 한도(limit) 한도 설정

```yaml
kind: ResourceQuota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "10"
    limits.memory: "10Gi"
```

- 쿼터 초과 시 더 이상 Pod 생성 불가

## Pod, Deployment resource 편집
### Pod 편집
- 직접 수정 가능한 필드
  - `spec.containers[*].image`
  - `spec.initContainers[*].image`
  - `spec.activeDeadlineSeconds`
  - `spec.tolerations`
- 수정 불가 필드 예시
  - 환경 변수, 서비스 계정, 리소스 요청/한도 등
- 불가능한 필드 변경 방법
  - `kubectl edit pod <pod>` 시도 → 저장 거부 → 임시 YAML 파일 생성
  - → `kubectl delete pod <pod>` → `kubectl create -f /tmp/kubectl-edit-*.yaml`
	- `kubectl get pod <pod> -o yaml > my-pod.yaml` → 파일 수동 편집
  - → `kubectl delete pod <pod> → kubectl create -f my-pod.yaml`

### Deployment 편집
- 모든 Pod 템플릿 필드 편집 가능
  - 환경 변수, 리소스 요청/한도 등 모두 수정 가능

```bash
kubectl edit deployment <deployment-name>
```

- 동작 방식
  - Deployment의 Pod 템플릿이 변경되면 롤링 업데이트로 기존 Pod를 삭제하고 새 Pod를 생성함.

## DaemonSets
### DaemonSet의 개념
- 클러스터의 각 노드마다 “Pod 한 개”를 항상 실행하도록 보장하는 리소스
- 노드가 추가되면 자동으로 해당 노드에도 Pod를 생성하고, 노드가 삭제되면 Pod를 제거

### ReplicaSet/Deployment와의 차이
- ReplicaSet은 원하는 복제본 수만큼 Pod를 띄우지만, DaemonSet은 “노드 수 = Pod 수”를 유지
- 신규 노드에는 수동 작업 없이 자동으로 Pod가 배포

### 주요 사용 사례
- 모니터링 에이전트 (예: Prometheus Node Exporter)
- 로그 수집기 (예: Fluentd, Filebeat)
- kube-proxy 같은 클러스터 워커 컴포넌트
- 네트워킹 플러그인 에이전트 (예: Calico, Weave Net)

### DaemonSet 매니페스트 구조
1. `apiVersion` / `kind: DaemonSet`
2. `metadata: { name: … }`
3. `spec`
	- `selector`: Pod 레이블 매칭
	- `template`: Pod 스펙 (레이블 + 컨테이너 사양)

```bash
kubectl create -f daemonset.yaml


kubectl get daemonset
kubectl describe daemonset <이름>
```

### 스케줄링 방식
- 초기(v1)에는 `nodeName` 필드를 직접 설정했으나
- 현재는 “기본 스케줄러 + 노드 선호도(Affinity/Anti-Affinity)”를 사용하여 자동 배치

## Static Pod
### Static Pod(정적 Pod)의 개념
- API 서버나 컨트롤러 없이, kubelet이 로컬에 놓인 매니페스트 파일을 직접 읽어 생성·관리하는 Pod
- 클러스터 외부에서 독립 실행될 수 있으며, kubelet이 “주인” 역할

### 동작 원리
1. `--pod-manifest-path`로 지정된 디렉터리(기본: `/etc/kubernetes/manifests`)를 kubelet이 주기적으로 스캔
2. 매니페스트 파일이 추가되면 Pod 생성, 수정되면 재생성, 삭제되면 Pod 제거
3. 애플리케이션 장애 시 kubelet이 자동 재시작 보장

### Mirror Pod
- Static Pod가 클러스터 일원으로 동작할 때 kubelet이 API 서버에 “읽기 전용 복제본”(Mirror Pod) 객체 등록
- `kubectl get pods --all-namespaces`에서 볼 수 있으나, 직접 수정·삭제는 불가

### 스케줄링
- 일반 Pod는 kube-scheduler가 노드를 결정하지만, Static Pod는 스케줄러를 거치지 않고 kubelet이 로컬에서 바로 스케줄링

### 주요 사용 사례
- Control Plane 컴포넌트(kube-apiserver, controller-manager, scheduler 등)를 각 노드에 Pod 형태로 배포
- 클러스터 설치 도구(kubeadm 등)가 Control Plane을 구성할 때 주로 활용

### Static Pod vs DaemonSet
- Static Pod: kubelet이 로컬 파일 기반으로 직접 관리
- DaemonSet: API 서버 + 컨트롤러가 클러스터 전역 노드마다 Pod를 배포·관리

### 설정 확인 및 실습
1. 노드에서 kubelet 옵션(`--pod-manifest-path`) 확인
2. 해당 디렉터리에 매니페스트 파일 배치 후 `docker ps` 또는 `kubectl get pods -n kube-system`로 생성 여부 점검
3. 파일 변경/삭제로 Pod 갱신·삭제 실습

## Priority Class
### PriorityClass의 역할
- 클러스터 내 서로 다른 중요도의 워크로드에 “숫자 기반 우선순위”를 부여
- 우선순위가 높은 Pod가 리소스 부족 상황에서도 반드시 스케줄될 수 있도록 보장
- 필요 시 낮은 우선순위 Pod를 종료(선점)하거나 대기시켜 리소스를 확보

### PriorityClass 객체
- `apiVersion`: `scheduling.k8s.io/v1`
- `kind`: `PriorityClass`
- 필수 필드
  - `metadata.name`: 클래스 이름
  - `value`: 우선순위 값(정수, 클러스터 워크로드는 –2,000,000,000 ~ +1,000,000,000 범위)
- 선택 필드
  - `globalDefault`: true/false (true인 클래스가 Pod에 지정된 우선순위가 없을 때 기본값으로 사용)
  - `description`: 설명 텍스트
  - `preemptionPolicy`: “PreemptLowerPriority”(기본) 또는 “Never”(선점 비활성화)
- 기본 우선순위
  - Pod에 `priorityClassName`을 지정하지 않으면 우선순위 0(값이 0인 PriorityClass)로 간주
  - 글로벌 기본 클래스 하나만 `globalDefault`: true로 설정 가능

## Multiple Schedulers
### 다중 스케줄러가 필요한 이유
- 기본 스케줄러가 제공하는 정책(Taint, Tolerance, Affinity 등)으로 충분하지 않을 때
- 특정 워크로드만을 위한 고유한 스케줄링 로직(예: 노드별 특수 조건) 적용을 위해

### 사용자 정의 스케줄러 제작 및 패키징
1. 스케줄러 바이너리
  - 기본 kube-scheduler 바이너리 사용 또는 직접 구현한 스케줄러 코드 패키징
2. 고유 이름 지정
  - `--scheduler-name=<my-scheduler>` 옵션으로 스케줄러 식별자 설정
  - 기본 스케줄러는 내부적으로 default-scheduler로 이름 지정

### HA(고가용성) 구성
- 마스터 노드 여러 대에서 동일한 스케줄러 복제본 실행
- `--leader-elect=true`, `--leader-election-id=<id>` 등으로 리더 선출 활성화

### 스케줄러 배포 방법
1. 독립 서비스로 실행
	- 바이너리 + 사용자 정의 kubeconfig + 스케줄러 구성 파일(config.yaml)
2. Pod/Deployment 형태로 배포
  - Deployment 매니페스트 작성
  - VolumeMount 또는 ConfigMap으로 스케줄러 설정(config.yaml) 주입
  - ServiceAccount, ClusterRole/ClusterRoleBinding 생성해 API 서버 인증·인가 처리

```yaml
args:
  - "--config=/etc/kube-scheduler/config.yaml"
  - "--scheduler-name=my-scheduler"
  - "--leader-elect=true"
volumeMounts:
  - name: config
    mountPath: /etc/kube-scheduler
volumes:
  - name: config
    configMap:
      name: my-scheduler-config
```

### 워크로드에 커스텀 스케줄러 지정
- Pod/Deployment 정의 시 `spec.schedulerName: my-scheduler` 추가
- 지정하지 않으면 기본 스케줄러에 할당

### 검증 및 문제해결
1. 스케줄러 상태 확인

```bash
kubectl get pods -n kube-system -l component=kube-scheduler
```

2. 워크로드 스케줄 확인

```bash
kubectl apply -f my-pod.yaml
kubectl get events -o wide
```

이벤트 SOURCE에 `my-scheduler` 이름이 표시됨

3. 스케줄러 로그 조회

```bash
kubectl logs deployment/my-scheduler -n kube-system
```

## Configuring Scheduler Profiles
### 스케줄러 파이프라인 단계
1. 큐(Queue) 단계: 대기 중인 Pod를 우선순위 기준으로 정렬(QueueSort 플러그인)
2. PreFilter → Filter 단계: 각 Pod가 실행 가능한 노드를 걸러냄
  - 예: `NodeResourcesFit`(리소스 요구), `NodeName`(특정 노드 지정), `NodeUnschedulable`(드레인된 노드 제외)
3. PostFilter → Score 단계: 필터 통과한 노드에 점수 부여
  - 예: `LeastRequested`(여유 리소스), `ImageLocality`(이미지 캐시)
4. Reserve → Permit → PreBind → Bind → PostBind 단계: 최종 노드에 Pod를 예약·승인·바인딩
  - 기본 바인딩은 `DefaultBinder` 플러그인 담당

### 플러그인 확장 포인트
- 각 단계마다 `preFilter`, `postFilter`, `score`, `reserve`, `permit`, `preBind`, `postBind` 등 다양한 확장 지점이 있어
- 사용자는 필요에 따라 표준 플러그인 활성화/비활성화하거나, 자체 플러그인을 개발해 삽입 가능

### 기존 다중 스케줄러의 문제
- 서로 다른 바이너리(프로세스)로 운영 → 관리 복잡, 스케줄 충돌 우려

### 스케줄러 프로필 도입(1.18, CAP 1451)
- 단일 kube-scheduler 바이너리 안에 여러 프로필(profiles 배열)을 정의
- 각 프로필에 고유한 `schedulerName`과 플러그인 설정(활성·비활성·가중치)을 지정
- 프로필별로 다른 스케줄링 로직·조건 적용 가능

### 사용 방법 요약
1. ConfigMap 또는 파일로 스케줄러 설정 작성
2. `profiles:` 항목에 프로필별 이름·플러그인 구성 추가
3. 스케줄러 실행 시 `--config`로 해당 설정 로드
4. Pod 정의에 `spec.schedulerName: <프로필명>` 지정 → 해당 프로필로 스케줄링

## Admission Controllers
### Admission Controller 개념
- API 서버에 도달한 요청은 인증(Authentication) → 권한 부여(Authorization) 단계를 거친 뒤, Admission Controller를 통해 최종 검증·변경·거부
- RBAC으로는 불가능한 “이미지 레지스트리 제한”, “루트 사용자 실행 금지”, “레이블 강제 삽입” 등 추가 정책 구현

### 주요 기능
- Validation: 요청된 오브젝트의 스펙 유효성 검사
- Mutation: 요청을 자동으로 수정하거나 기본값 추가(예: `alwaysPullImages`, 기본 스토리지 클래스 지정)
- Rate Limiting: API 서버 과부하 방지(`EventRateLimit`)
- 네임스페이스 검사·프로비저닝: 존재하지 않는 네임스페이스 요청 거부 또는 자동 생성

### 기본 내장 컨트롤러 예시
- `AlwaysPullImages`
- `DefaultStorageClass`
- `EventRateLimit`
- `NamespaceLifecycle` (구 `NamespaceExists` + 자동 프로비저닝 대체)

### Admission Controller 활성화/비활성화
- kube-apiserver 실행 시 `--enable-admission-plugins/--disable-admission-plugins` 플래그로 설정
- kubeadm 환경: `/etc/kubernetes/manifests/kube-apiserver.yaml` 내 `command:` 섹션에 해당 플래그 추가 또는 수정

### 네임스페이스 자동 프로비저닝 사례
1. 네임스페이스가 없는 상태에서 파드 생성 요청
2. Admission Controller(`NamespaceLifecycle` 이전엔 `NamespaceAutoProvision`)가 요청 감지
3. 네임스페이스 자동 생성 후 파드 생성 완료

현재 활성화된 Admission Controller 리스트 확인

```bash
kubectl exec -n kube-system kube-apiserver-<pod> -- \
  kube-apiserver -h | grep admission
```

## Validating and Mutating Admission Controllers
### 내장(In-tree) 컨트롤러 예시
- 유효성 검사(Validating)
  - NamespaceExists / NamespaceLifecycle: 네임스페이스 존재 여부 검사·거부
- Mutating
  - DefaultStorageClass: PVC에 스토리지 클래스 미지정 시 기본값 삽입
  - AlwaysPullImages, EventRateLimit 등

### Mutating vs Validating 순서
1. MutatingAdmissionController 실행 → 객체 스펙 변경
2. ValidatingAdmissionController 실행 → 변경된 스펙 검증·허용/거부

### 커스텀 Admission Webhook
- 외부(또는 클러스터 내) 서버에서 MutatingAdmissionWebhook / ValidatingAdmissionWebhook 구현 가능
- AdmissionReview 객체(JSON)로 요청 정보(사용자, 작업, 리소스, 오브젝트 등) 전달
- 서버는 AdmissionReview 응답(allowed: true/false, 또는 JSON Patch)을 반환

### 설정 단계
1. 웹훅 서버 개발·배포
  - Go, Python 등 언어로 AdmissionReview API 핸들러 구현
  - 사용자 요청에 따라 허용/거부 또는 JSON Patch 생성
  - 클러스터 내 Deployment + Service로 배포하거나 외부에서 호스팅
2. TLS 인증 구성
  - 웹훅 서버용 인증서·키 준비
  - caBundle에 CA 인증서(base64) 설정
3. WebhookConfiguration 생성

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration  # MutatingWebhookConfiguration도 비슷
metadata:
  name: my-validating-webhook
webhooks:
  - name: pod-policy.example.com
    clientConfig:
      service:
        name: webhook-service
        namespace: my-namespace
        path: "/validate"
      caBundle: <base64-encoded-CA>
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
```

4. 운영
  - 지정된 리소스·작업에 대해 API 서버가 `AdmissionReview` 호출
  - 웹훅 서버 응답에 따라 요청이 허용되거나 거부
  - Mutating 웹훅의 경우 JSON Patch로 오브젝트 자동 수정


사용자 레이블 삽입 Mutating Webhook 예시
```python
def mutate(admission_request):
    user = admission_request.userInfo.username
    patch = [{
      "op": "add",
      "path": "/metadata/labels/username",
      "value": user
    }]
    return AdmissionResponse(allowed=True, patch=base64_encode(json.dumps(patch)))
```
