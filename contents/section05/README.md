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
- 기존 파드를 모두 종료한 뒤 새 버전 파드 일괄 생성
- 다운타임 발생
2. RollingUpdate (기본 전략)
- 구 버전 파드를 점진적으로 축소(terminate)하면서 새 버전 파드를 순차 생성
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