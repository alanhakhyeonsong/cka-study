# Application Lifecycle Management
## Rolling Updates and Rollbacks
### 롤아웃(Deployment Rollout)과 리비전(Revision)
- 최초 배포 생성 시 자동으로 리비전 1(Revision 1) 롤아웃 발생
- 업데이트(컨테이너 이미지 변경 등) 시마다 새로운 리비전(Revision 2, 3, …)이 생성되어 변경 내역을 추적

### 롤아웃 상태 및 기록 조회
- `kubectl rollout status deployment/<deployment-name>`
- 현재 롤아웃 진행 상황(성공·실패·진행 중) 확인
- `kubectl rollout history deployment/<deployment-name>`
- 각 리비전의 생성 시점 및 변경된 필드(이미지, 레플리카 수 등) 확인

### 배포 전략(Deployment Strategy)
1. Recreate
- 기존 Pod를 모두 종료한 뒤 새 버전 Pod 일괄 생성
- 다운타임 발생
2. RollingUpdate (기본 전략)
- 구 버전 Pod를 점진적으로 축소(terminate)하면서 새 버전 Pod를 순차 생성
- 무중단(up-and-running) 상태로 버전 전환 가능

### deployment 업데이트 방법
- YAML 수정 후 적용

```bash
kubectl apply -f deployment.yaml
```

- kubectl set image

```bash
kubectl set image deployment/<deployment-name> <컨테이너명>=<새이미지>
```

- 즉시 이미지 버전만 교체하며, 리비전이 자동 생성됨
- 선언적 정의 파일에도 동일 변경을 반영 필요

### 롤백(Rollback)
- 문제 발생 시 이전 리비전으로 되돌리기

```bash
kubectl rollout undo deployment/<deployment-name>
```

- 롤백 전후의 ReplicaSet·Pod 상태 변화를 `kubectl get rs` 명령으로 비교 가능

### 자주 쓰는 명령어 모음
- deployment 생성

```bash
kubectl create deployment <이름> --image=<이미지>
```

- 롤아웃 상태 확인

```bash
kubectl rollout status deployment/<deployment-name>
```

- 롤아웃 기록 조회

```bash
kubectl rollout history deployment/<deployment-name>
```

- 롤백

```bash
kubectl rollout undo deployment/<deployment-name>
```

## Commands and Arguments in Docker
### 컨테이너 실행 기본 개념
- 컨테이너는 운영체제 전체를 호스팅하지 않고, 특정 프로세스/작업만 실행.
- 컨테이너 안의 주 프로세스가 종료되면 컨테이너도 종료됨.
- 예: Ubuntu 이미지는 기본 명령이 bash인데, 터미널이 연결되지 않으면 즉시 종료됨.

### CMD (명령)
- Dockerfile에서 **CMD 지시어는 컨테이너 시작 시 실행할 기본 명령을 정의.**
- 예
  - nginx 이미지 → nginx 실행
  - mysql 이미지 → mysqld 실행
- `docker run <image> <command>` 로 실행 시 CMD를 재정의(override) 할 수 있음.
- 형식: 문자열(shell form) 또는 JSON 배열(exec form) → exec form 권장.

### ENTRYPOINT (진입점)
- ENTRYPOINT는 **컨테이너가 항상 실행할 프로그램을 지정.**
- CMD와의 차이
  - CMD: 실행할 명령 전체를 바꿀 수 있음.
  - ENTRYPOINT: 실행할 명령은 고정, 커맨드라인 인수만 추가됨.
- 예
  - ENTRYPOINT: ["sleep"]
  - CMD: ["5"]
  - 실행 결과: `sleep 5`
  - `docker run myimage 10` → `sleep 10`

### CMD와 ENTRYPOINT의 조합
- ENTRYPOINT + CMD → 기본 프로그램과 기본 인수 지정 가능.
  - ENTRYPOINT는 고정 실행 파일
  - CMD는 기본 인수, 필요 시 커맨드라인에서 무효화 가능
- 항상 JSON(exec form) 사용 권장 (쉘 파싱 문제 방지).

### 런타임 재정의
- ENTRYPOINT 자체를 바꾸고 싶으면 `docker run --entrypoint` 옵션 사용.

### 정리
- `CMD`: 기본 명령, 실행 시 전체를 바꿀 수 있음.
- `ENTRYPOINT`: 항상 실행할 명령, 인수만 바꿀 수 있음.
- ENTRYPOINT + CMD를 함께 사용하면 “고정 명령 + 기본 인수” 조합 가능.

## Commands and Arguments in Kubernetes
### Kubernetes Pod 정의에서의 매핑
- Pod 정의 파일에서 command와 args를 통해 Dockerfile의 ENTRYPOINT / CMD를 제어할 수 있음.
- args: Dockerfile의 CMD 값을 재정의 (기본 매개변수 변경)
- command: Dockerfile의 ENTRYPOINT 값을 재정의 (실행 명령 자체 변경)

### 동작 원리
- Dockerfile의 ENTRYPOINT ↔ Pod spec의 command
- Dockerfile의 CMD ↔ Pod spec의 args
- `args` = 기본 인수 변경 (예: sleep 시간을 5초 → 10초)
- `command` = 실행 명령 자체 변경 (예: ENTRYPOINT sleep → sleep2)

### 핵심 정리
- Pod 정의에서
  - command: ENTRYPOINT 무효화
  - args: CMD 무효화
- 이 둘을 통해 컨테이너 실행 시 어떤 명령과 인수를 사용할지 유연하게 제어 가능.

## Configure Environment Variables in Applications
### 기본 방법
- Pod 정의 파일에서 env 속성을 사용.
- env는 배열 형태이며 각 항목은 name(환경 변수 이름)과 value(값)으로 구성.
- 지정된 환경 변수는 컨테이너 내부에서 사용 가능.

```yaml
env:
- name: APP_MODE
  value: "production"
```

### 다른 방법
- ConfigMap 또는 Secret에서 값을 가져와 환경 변수로 주입 가능.
- 이때는 `value` 대신 `valueFrom` 속성을 사용.

```yaml
env:
- name: APP_MODE
  valueFrom:
    configMapKeyRef:
    # secretKeyRef:
```

### 핵심 정리
- 직접 key-value 지정 가능.
- 또는 ConfigMap / Secret을 통해 보다 안전하고 관리 가능한 방식으로 설정 가능.

## Configuring ConfigMaps in Applications
### ConfigMap 개념
- ConfigMap은 Kubernetes에서 키-값 쌍의 구성 데이터를 전달하는 객체.
- 애플리케이션 코드와 설정을 분리하여 관리 가능.
- 포드가 생성될 때 ConfigMap 데이터를 환경 변수나 파일로 주입 가능.

### ConfigMap 생성 방법
1. 명령형(Imperative) 방법

- 리터럴 값 지정

```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod
```

- 파일에서 읽기

```bash
kubectl create configmap app-config --from-file=app.properties
```

2. 선언형(Declarative) 방법 (YAML 정의 파일)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```

→ `kubectl apply -f configmap.yaml`

### ConfigMap 확인
- `kubectl get configmap` : ConfigMap 목록 확인
- `kubectl describe configmap app-config` : 데이터 확인

### Pod에서 사용하기
1. 환경 변수로 주입

```yaml
envFrom:
- configMapRef:
    name: app-config
```

→ ConfigMap의 모든 key/value가 컨테이너 환경 변수로 전달됨.

2. 특정 환경 변수로 주입

```yaml
env:
- name: APP_COLOR
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_COLOR
```

3. 볼륨으로 마운트 (파일 형태)

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
```

### 정리
- ConfigMap = 설정 데이터 중앙 관리 도구
- 생성 방법: 명령형(리터럴/파일) + 선언형(YAML)
- 사용 방법: Pod에 env, envFrom, volume 으로 주입
- 요약하면, ConfigMap을 활용하면 여러 Pod에 공통 설정을 중앙에서 관리하고 손쉽게 주입할 수 있다는 내용.

## Secrets
### Secret의 필요성
- ConfigMap은 일반 텍스트라서 비밀번호/민감정보 저장에는 적합하지 않음.
  - Secret을 사용하여 비밀번호, 토큰, 키 같은 민감정보를 저장.
- Secret은 etcd에 base64 인코딩 형태로 저장됨 (단, 암호화는 아님).

### Secret 생성 방법
1. 명령형 (Imperative)

- 리터럴 값으로 생성

```bash
kubectl create secret generic app-secret \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=mypassword
```

- 파일에서 생성

```bash
kubectl create secret generic app-secret \
  --from-file=./db-password.txt
```

2. 선언형 (Declarative, YAML 파일)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_HOST: bXlzcWw=        # base64(mysql)
  DB_USER: YWRtaW4=        # base64(admin)
  DB_PASS: bXlwYXNzd29yZA== # base64(mypassword)
```

값은 반드시 base64 인코딩 필요.

### Secret 확인
- `kubectl get secret` : Secret 목록 확인
- `kubectl describe secret app-secret` : 메타데이터만 표시 (값은 숨겨짐)
- `kubectl get secret app-secret -o yaml` : base64 인코딩 값 확인 가능
- `echo "bXlwYXNzd29yZA==" | base64 --decode` 로 복호화 가능

### Pod에서 Secret 사용 방법
1. 환경 변수로 주입

```yaml
envFrom:
- secretRef:
    name: app-secret
```

→ Secret key들이 환경 변수로 사용 가능 또는 특정 키만 매핑

```yaml
env:
- name: DB_PASS
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: DB_PASS
```

2. 볼륨으로 주입 (파일 형태)

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: app-secret
```

→ Secret의 각 key가 파일 이름, 값이 파일 내용으로 저장됨.

### 보안 유의사항
- Secret은 기본적으로 암호화되지 않고 base64 인코딩만 됨 → 디코딩하면 바로 평문 확인 가능.
- 따라서
  - Secret YAML을 GitHub 등 버전 관리에 포함 X
  - etcd at-rest encryption 설정 필요 (EncryptionConfiguration 사용).
  - Secret 접근은 RBAC(Role-Based Access Control) 으로 제한해야 함.
  - 가능하다면 외부 Secret 관리 서비스 사용 (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, HashiCorp Vault 등).

### 정리
- ConfigMap: 일반 설정 데이터 → 민감정보 저장 X
- Secret: 민감정보 저장용 객체 (base64 인코딩)
- Pod에서 환경 변수나 볼륨으로 주입 가능
- 보안을 위해 RBAC, etcd 암호화, 외부 비밀 관리 솔루션을 반드시 고려해야 함

## Multi Container Pods
### 마이크로서비스와 모놀리식의 차이
- 대규모 모놀리식을 작은 마이크로서비스로 분리 → 독립적 개발·배포 가능.
- 필요 시 개별 서비스만 수정/확장 가능.

### 멀티 컨테이너 Pod의 필요성
- 경우에 따라 두 서비스가 항상 함께 동작해야 함 (예: 웹 서버 + 메인 앱).
- 서비스 코드를 병합하지 않고도 같은 생명주기를 공유하도록 구성.

### 멀티 컨테이너 Pod 특징
- 동일한 라이프사이클 공유 (함께 생성/종료).
- 네트워크 공간 공유 → localhost 로 서로 통신 가능.
- 스토리지 볼륨 공유 가능.
- Pod 간 별도 Service/Volume 없이 컨테이너끼리 직접 통신 가능.

### 구현 방식
- Pod 정의 파일의 `spec.containers` 는 배열 형태 → 여러 컨테이너 정의 가능.
- 예시
  - 컨테이너 1 → web-app
  - 컨테이너 2 → main-app

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    app: demo-app
spec:
  containers:
  - name: web-app
    image: nginx:1.27
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: main-app
    image: busybox:latest
    command: ["sh", "-c"]
    args:
    - |
      echo "<h1>Hello from Main App</h1>" > /usr/share/nginx/html/index.html;
      sleep 3600
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: shared-data
    emptyDir: {}
```

멀티 컨테이너 Pod는 서로 밀접하게 연관된 서비스(웹 서버 + 앱 등)를 같은 Pod에 배치해 동일한 네트워크·스토리지·라이프사이클을 공유하도록 하는 방법이다.

## Multi Container Pods Design Patterns
### 공동 배치된 컨테이너 (Co-located Containers)
- 가장 단순한 멀티 컨테이너 형태.
- Pod의 `spec.containers` 배열에 여러 컨테이너 정의.
- 특징
  - 모든 컨테이너가 동시에 시작되고 종료됨.
  - 시작 순서를 지정할 수 없음.
  - 서로 종속적인 두 서비스가 항상 같이 실행되어야 할 때 사용.

### 초기화 컨테이너 (Init Containers)
- Pod가 시작될 때 **먼저 실행되는 준비용 컨테이너.**
- 작업 완료 후 종료되며, 그 다음에 본 컨테이너(App 컨테이너)가 시작됨.
- 여러 개 정의 가능 → **순차적으로 실행됨.**
- 사용 예
  - DB 준비가 될 때까지 대기
  - 다른 API 서비스가 준비될 때까지 검사

### 사이드카 컨테이너 (Sidecar Containers)
- **Init Container처럼 먼저 실행되지만, 종료되지 않고 Pod 생명주기 동안 계속 실행됨.**
- 메인 앱과 함께 실행되며, 앱이 중지된 후에도 종료 로그 등을 캡처 가능.
- 사용 예
  - 로그 수집기 (Filebeat) → Elasticsearch/Kibana로 로그 전달
  - 메인 애플리케이션 실행 전/중/후 로그를 모두 수집 가능

### 정리
- 공동 배치된 컨테이너: 단순히 함께 실행, 시작 순서 없음.
- 초기화 컨테이너: 본 앱 실행 전에 반드시 실행되어 종료되는 준비 작업.
- 사이드카 컨테이너: 본 앱 실행 전 시작 → 앱과 함께 계속 실행 → 앱 종료 시까지 동작.

## Init Containers
### 멀티 컨테이너 Pod의 일반적인 동작
- Pod 안의 각 컨테이너는 Pod 생명주기와 함께 계속 살아 있어야 함.
- 예: 웹 애플리케이션 컨테이너 + 로그 수집 에이전트 컨테이너 → 둘 다 항상 실행 상태 유지.
- 둘 중 하나라도 실패하면 Pod 전체가 재시작됨.

### 일회성 작업이 필요한 경우
- 가끔은 컨테이너가 한 번만 실행되고 종료되기를 원할 수 있음.
- 예시
  - 애플리케이션 실행 전에 필요한 코드/바이너리를 외부 저장소에서 가져오는 작업
  - 외부 서비스(DB, API 등)가 준비될 때까지 기다리는 작업

### Init Container의 개념
- 이런 경우 Init Container 사용.
- 일반 컨테이너와 비슷하게 정의하지만, Pod의 spec.initContainers 항목에 선언.
- 특징
  - Pod가 시작될 때 먼저 실행되며, 반드시 완료(성공)해야 본 컨테이너들이 실행됨.
  - 여러 개를 정의하면 순차적으로 실행됨.
  - 하나라도 실패하면 Kubernetes가 Pod를 반복해서 재시작 → 성공할 때까지 시도.

### 예시
- 코드 클론을 먼저 실행하는 Init Container

```yaml
initContainers:
- name: init-myservice
  image: busybox
  command: ['sh', '-c', 'git clone <repo> ; done;']
```

- 외부 서비스 준비 대기 Init Container

```yaml
initContainers:
- name: init-myservice
  image: busybox:1.28
  command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
- name: init-mydb
  image: busybox:1.28
  command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

## Introduction to Autoscaling
### 스케일링 기본 개념
- 수직 확장(Vertical Scaling): 기존 서버/노드에 CPU·메모리 같은 리소스를 더 추가하는 방식.
- 수평 확장(Horizontal Scaling): 서버나 인스턴스를 추가해서 부하를 분산하는 방식.

### Kubernetes에서의 스케일링
- 클러스터 스케일링
  - 수평 확장: 노드를 더 추가.
  - 수직 확장: 기존 노드의 리소스를 늘림 (비교적 드문 방식).
- 워크로드 스케일링
  - 수평 확장: 더 많은 Pod를 생성.
  - 수직 확장: Pod에 할당된 리소스를 늘림.

### 스케일링 방식
- 수동 스케일링
  - 클러스터: 새 노드 프로비저닝 후 `kubeadm join` 으로 클러스터에 추가.
  - 워크로드: `kubectl scale` 로 Pod 개수 조정.
  - Pod 리소스: `kubectl edit` 로 리소스 요청/제한 변경.
- 자동 스케일링
  - Cluster Autoscaler: 인프라(노드)를 자동으로 확장/축소.
  - Horizontal Pod Autoscaler (HPA): Pod 개수를 자동으로 조절.
  - Vertical Pod Autoscaler (VPA): Pod 리소스 요청/제한을 자동으로 조절.

## Horizontal Pod Autoscaler (HPA)
### 수동 확장
- `kubectl top pod` 로 리소스 사용량(CPU/메모리) 확인. (→ Metrics Server 필요)
- 특정 임계값(예: CPU 450m)에 도달하면 `kubectl scale` 명령어로 Pod 수를 늘림.
- 문제점: 관리자가 직접 모니터링해야 하고, 급격한 트래픽 변화에 빠르게 대응 불가.

### 자동 확장(HPA)
- HPA는 메트릭 서버를 통해 Pod 리소스 사용량을 지속적으로 모니터링.
- CPU/메모리 또는 사용자 정의 메트릭 기준으로 자동으로 Pod 수를 조절.
- 사용량이 증가 → Pod 추가
- 사용량이 감소 → Pod 축소
- 최소/최대 Pod 개수 범위를 유지하면서 확장/축소 수행.

### HPA 생성 방법
- 명령형 방식

```bash
kubectl autoscale deployment my-app \
  --min=1 --max=10 \
  --cpu-percent=50
```
→ CPU 사용률이 50% 넘으면 스케일 아웃.

- 선언형 방식

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### HPA 동작 확인
- `kubectl get hpa` 로 확인 가능:
  - 현재 CPU 사용량
  - 목표 사용률
  - 최소/최대 Pod 수
  - 현재 복제본 수
- 필요 없으면 `kubectl delete hpa` 로 삭제 가능.

### 확장 메트릭 소스
- 기본: Metrics Server (CPU, 메모리 사용량)
- 사용자 정의 메트릭 어댑터: 클러스터 내부 앱에서 제공하는 지표
- 외부 메트릭 어댑터: Datadog, Dynatrace 등 외부 모니터링 도구

HPA는 Pod의 CPU/메모리 사용량을 기준으로 자동 확장을 담당하며, **시험에서는 Metrics Server + HPA YAML 정의 정도를 이해**하는 게 핵심

## In-place Resize of Pods
### 기존 동작
- Kubernetes v1.32 기준
  - Pod 리소스 요청/제한(CPU, 메모리 등)을 변경하면 기존 Pod를 종료하고 새 Pod를 생성해야 함.
  - 즉, 인플레이스 업데이트 불가.

### 인플레이스 크기 조정 (In-Place Pod Vertical Scaling)
- Kubernetes v1.27 알파 기능으로 도입 → 이후 v1.33 알파 확장됨.
- InPlacePodVerticalScaling FeatureGate 활성화 시 사용 가능.
- 컨테이너를 종료하지 않고 CPU/메모리 리소스를 조정할 수 있음.
  - 예: CPU 변경 시 재시작 불필요, 메모리 변경 시 재시작 필요하도록 정책 설정 가능.

### 크기 조정 정책
-	리소스별로 재시작 여부 지정 가능.
  -	CPU → 재시작 없이 업데이트
  -	메모리 → 재시작 필요

### 제한 사항
- 지원 범위: CPU, 메모리 리소스만 가능
- 변경 불가 사항
  - Pod QoS 클래스
  - Init 컨테이너 및 Ephemeral 컨테이너
- 메모리 관련 제약
    - 메모리 제한은 사용량 이하로 줄일 수 없음
  - 줄이려 할 경우 → 조건 충족 전까지 “Resizing in-progress” 상태 유지
- Windows Pod는 현재 지원되지 않음

## Vertical Pod Autoscaling (VPA)
### 수동 수직 확장
- 관리자가 `kubectl top pod`로 리소스 모니터링 후 `kubectl edit deployment`로 CPU/메모리 요청·제한 변경.
- 기존 Pod 종료 → 새 Pod 생성.
- 수동 관리 방식 → 비효율적.

### VPA
- HPA와 달리 쿠버네티스에 기본 내장되지 않음 → 별도 설치 필요.
- 주요 컴포넌트
  - 추천자(Recommender): CPU/메모리 사용량 분석 → 권장 리소스 값 산출.
  - 업데이터(Updater): 최적치 벗어난 Pod 종료(퇴출).
  - 어드미션 컨트롤러(Admission Controller): 새 Pod 생성 시 권장 리소스 적용.

### 동작 방식
1. 추천자가 메트릭 수집 → 권장 값 계산.
2. 업데이터가 현재 Pod와 비교 → 필요 시 Pod 종료.
3. 어드미션 컨트롤러가 새 Pod 기동 시 권장 값 반영.

### 업데이트 정책 모드
- Off: 권장만 제공, 실제 변경 없음.
- Initial: 최초 생성 시만 권장 값 적용.
- Recreate: 필요 시 Pod 종료 후 새로 생성.
- Auto: 현재는 Recreate와 동일, 향후 In-Place 업데이트 지원 시 자동 적용 예정.

### HPA vs VPA

| **구분** | **HPA (수평 확장)** | **VPA (수직 확장)** |
| --- | --- | --- |
| 방식 | Pod 수 증감 | 개별 Pod 리소스 조정 |
| Pod 동작 | 기존 Pod 유지 + 새 Pod 생성 → 무중단 | Pod 재시작 필요 → 다운타임 발생 |
| 트래픽 폭증 대응 | 빠른 확장 가능 → 적합 | 대응 느림 |
| 비용 최적화 | 유휴 Pod 제거 | 리소스 과할당 방지 |
| 적합 워크로드 | 웹 서버, 마이크로서비스, API | DB, JVM 앱, AI/ML 등 CPU·메모리 집약형 |

### 활용 시나리오
- VPA 적합: 상태 저장 워크로드, DB, 초기화 시 CPU 많이 쓰는 애플리케이션.
- HPA 적합: 변동 트래픽 많은 상태 비저장 서비스.
- 두 가지를 병행할 수도 있음.