# Kustomize Basics
## Kustomize Problem Statement & idealogy
- 여러 환경(개발, 스테이징, 프로덕션)에서 Kubernetes 배포를 환경별로 다르게 설정하고 싶다.
	-	예시: nginx 배포에서
	-	개발: 복제본 1
	-	스테이징: 복제본 2~3
	-	프로덕션: 복제본 5~10

가장 단순한(하지만 문제 있는) 접근
-	환경별로 세 개의 디렉토리(dev, staging, prod)를 만들어서
  -	같은 리소스 정의를 복사하고
  -	각 환경에서 복제본만 다르게 설정
-	문제점
  -	서비스가 늘어나면 파일을 매번 각 환경 디렉토리에 복사해야 함
  -	모든 환경에서 동일한 변경을 일일이 반영해야 함
  -	실수나 누락이 발생하기 쉬움
  -	유지보수성/확장성 부족

### Kustomize의 등장 이유
- DRY(Don’t Repeat Yourself) 원칙 적용
-	환경별 변경이 필요한 부분만 정의해서 재사용성 높임
-	유지보수성과 확장성을 개선

### Kustomize 핵심 개념
- Base
  - 모든 환경에서 공통으로 사용하는 구성
  - eg. nginx 배포 정의 (기본 복제본 수 1)
- Overlay
  - 각 환경별로 Base를 덮어써서 커스터마이즈
  - eg
    - dev : Base 유지
    - staging: 복제본 2로 덮어쓰기
    - prod: 복제본 5로 덮어쓰기

```
.
├── base/
│   └── deployment.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml (optional)
    ├── staging/
    │   └── kustomization.yaml (replicas: 2)
    └── prod/
        └── kustomization.yaml (replicas: 5)
```

<img width="720" height="212" alt="Image" src="https://github.com/user-attachments/assets/89338edd-ab98-4c36-901b-aa5b868b4c58" />

### Kustomize의 장점
-	환경별로 달라지는 일부 속성만 정의
-	모든 구성은 표준 YAML → 특별한 템플릿 언어 필요 없음
-	kubectl에 내장되어 있어 추가 설치 없이 사용 가능 (단, 최신 기능 위해 별도 설치 권장)
-	단순하고 읽기 쉬움 → 복잡한 Helm 템플릿보다 가독성 높음

→ **환경별 구성을 복제하지 않고, 공통은 Base로 두고 환경별 차이는 Overlay로 관리해 유지보수성과 재사용성을 확보하자.**

## Kustomize vs Helm
### Helm이 문제를 해결하는 방식
-	Go 템플릿 구문 사용
  -	Kubernetes 매니페스트의 값에 변수를 삽입
  -	예: `replicas: {{ .Values.replicaCount }}`
-	환경별 값을 values.yaml 파일에 정의
  -	dev, staging, prod 각각 다른 values 파일 작성
  -	배포 시 템플릿에 값이 삽입되어 환경별로 다른 매니페스트 생성

```
chart/
├── templates/
│   └── deployment.yaml  (Go 템플릿 구문 사용)
└── values/
    ├── dev-values.yaml
    ├── staging-values.yaml
    └── prod-values.yaml
```

-	templates/에는 변수화된 리소스 정의
-	values/에는 환경별 변수 값 정의

### Helm의 특징
-	단순히 환경별 커스터마이즈뿐만 아니라 **Kubernetes용 패키지 관리자**
  -	리눅스의 APT/YUM과 유사한 역할
  -	차트(패키지)를 설치, 업그레이드, 버전 관리
-	추가 기능 지원
  -	조건문, 루프, 함수, 후크 등
  -	Kustomize에는 없는 강력한 기능 제공

### Helm의 단점
-	Go 템플릿 언어 사용 → 읽기 어려움
  -	유효한 YAML이 아니므로 가독성 저하
  -	템플릿이 복잡해지면 무엇을 배포하는지 파악이 어려움

### Kustomize와 Helm 비교
|항목|Kustomize|Helm|
|--|--|--|
|주된 특징|순수 YAML 기반 오버레이|Go 템플릿 기반 패키지 관리|
|학습 곡선|단순하고 쉽다|템플릿 언어 필요 → 조금 더 복잡|
|가독성|매우 높음|템플릿화로 인해 낮아질 수 있음|
|기능|환경별 차이 덮어쓰기|조건문, 루프, 후크 등 다양한 기능 지원|
|설치|kubectl에 내장(별도 설치 불필요)|별도 설치 필요(helm CLI)|

## Installation/Setup
### 사전 준비
-	Kustomize 설치 전:
  -	로컬 머신에 Kubernetes 클러스터가 실행 중이어야 함
  -	kubectl이 설치되어 있고 클러스터와 연결되어 있어야 함

### 설치 방법
-	Kustomize는 Linux, Windows, Mac에서 설치 가능
-	운영체제를 자동 감지해서 적절한 버전을 설치하는 공식 스크립트 제공
-	터미널에서 스크립트 한 줄 실행 → 자동 설치

### 설치 확인
-	설치 후
  -	`kustomize version` 명령으로 설치 여부 확인
  -	출력이 제대로 나오지 않으면:
    -	환경변수 문제일 수 있음 → 터미널 재시작 권장
    -	여전히 안되면 설치 스크립트 재실행

## Kustomization.yaml file
### 폴더/파일 구조
-	모든 Kubernetes 매니페스트(YAML)를 담는 k8s 디렉터리 예시
  -	nginx 배포 YAML
  -	nginx 서비스 YAML
-	Kustomize가 자동으로 인식하는 파일: `kustomization.yaml`
  -	반드시 직접 작성해야 함

### kustomization.yaml의 구성
- `resources`
	-	Kustomize가 관리할 모든 YAML 파일 나열
	-	예: nginx deployment와 service
- `transformations` / `customization`
	-	변형이나 커스터마이즈할 내용
	-	예: 공통 레이블 추가 (회사: KodeKloud)

### 간단한 예제
-	리소스
  -	nginx deployment
  -	nginx service
-	변형
  -	모든 리소스에 label 추가
    -	company: KodeKloud
-	결과
  -	두 리소스 모두 label이 추가됨

```
k8s/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
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
        image: nginx:1.21
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  company: KodeKloud
```

### 빌드 명령
-	`kustomize build [디렉터리]`
  -	지정 디렉터리(k8s)의 `kustomization.yaml`을 읽음
  -	`resources` 목록의 매니페스트들을 합침
  -	정의된 변형(예: 레이블 추가) 적용
  -	최종 YAML 출력 생성
-	주의
  -	`kubectl apply`를 자동으로 실행하지 않음
  -	단순히 결과 YAML을 출력 → 실제 배포는 별도로 수행

## Kustomize Output
-	`kustomize build` 명령
	-	모든 리소스와 변환을 합쳐서 최종 YAML 출력 생성
	-	Kubernetes에 적용하지는 않음
	-	→ 출력만 콘솔에 보여줌

### 배포
- 출력된 YAML을 `kubectl apply`로 넘겨야 실제 클러스터에 적용됨
-	일반적인 적용 명령 예

`kustomize build k8s/ | kubectl apply -f -`

-	`kustomize build k8s/` → 최종 매니페스트 생성
-	`|`(파이프) → 출력을 다음 명령으로 전달
-	`kubectl apply -f -` → 표준 입력(-)에서 받은 YAML을 클러스터에 적용

### kubectl -k 방식 배포
- `kubectl`은 `-k` 옵션으로 Kustomize 내장 지원

`kubectl apply -k k8s/`
- `-k`는 지정한 디렉토리의 kustomization.yaml을 자동 처리

### 삭제(제거) 방법
- 적용과 거의 동일 → `apply` → `delete`로 변경  
  - `kustomize build k8s/ | kubectl delete -f -`
  - 파이프를 통해 빌드한 리소스 정의를 삭제 명령에 전달
- kubectl 내장 방식
  - `kubectl delete -k k8s/`

## Kustomize ApiVersion & Kind
- `apiVersion`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
```
- kustomize 팀이 만든 kustomization CRD(Custom Resource Definition)의 버전
- Kustomize CLI가 이 버전을 읽고 올바르게 파싱함

- `kind`
  - 리소스의 종류 명시

```yaml
kind: Kustomization
```

- 이 YAML 파일이 Kustomize의 설정을 정의하는 파일이다 라는 선언
- Kustomize CLI가 이 파일을 **설정의 진입점**으로 인식

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  app: myapp
```

-	`apiVersion`이 맞아야 Kustomize가 이해함
-	`kind: Kustomization`이 있어야 “이건 Kustomize 설정”이라고 인식
-	이후 `resources`와 `commonLabels` 등의 세부 설정이 적용됨


## Managing Directories
-	초기에는 k8s 디렉터리에 여러 YAML 파일(API 배포, 서비스, DB 배포, 서비스 등)을 평면적으로 두고 `kubectl apply -f k8s/`로 한 번에 배포
-	파일 수가 늘어나면 관리가 복잡해지고 폴더를 분리하게 됨

```
k8s/
  api/
    deployment.yaml
    service.yaml
  database/
    deployment.yaml
    service.yaml
```

### 문제점
하위 디렉터리로 나눈다면,

- 각 디렉터리별로 따로 들어가서 kubectl apply -f를 여러 번 실행해야 함
- 디렉터리가 많아지면 매우 번거로움
- CI/CD 파이프라인도 각 디렉터리를 반복 처리해야 함

### Kustomize의 첫 번째 해결
- 루트 디렉터리에 kustomization.yaml 추가
- 이 파일에서 하위 디렉터리 경로를 모두 나열

```yaml
resources:
  - api/
  - database/
```

`kustomize build k8s/ | kubectl apply -f -` or `kubectl apply -k k8s/`

-	한 번의 명령어로 모든 하위 디렉터리 적용 가능
-	각 하위 디렉터리에 일일이 들어갈 필요 없음

### 두 번째 문제 - 스케일 문제
-	디렉터리가 2개 → 4개, 10개, 20개로 늘어나면?
-	루트 `kustomization.yaml`의 `resources` 섹션이 너무 길어지고 지저분해짐
-	수십~수백 개의 리소스 경로를 나열해야 함

### Kustomize의 두 번째 해결 (계층적 구성)
- 각 하위 디렉터리에도 자체 `kustomization.yaml` 작성

```yaml
# api/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml

---
# database/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml

---
# 루트(k8s/kustomization.yaml)는 하위 디렉터리만 나열
resources:
  - api/
  - database/
  - cache/
  - kafka/
```

`kustomize build k8s/ | kubectl apply -f -` or `kubectl apply -k k8s/`

-	각 폴더는 **자기 리소스만 관리**
-	루트는 **하위 폴더를 포함만** → 깔끔하고 유지보수성 향상
-	수백 개의 리소스를 계층적이고 모듈화된 형태로 관리 가능
- 루트 디렉터리에서 한 번의 명령으로 모든 리소스 배포
- CI/CD 파이프라인도 단일 단계로 관리 가능

## Patches Intro
### Patch의 개념
- Transformer: 전체 오브젝트에 전반적인 수정 적용 (예: 네임스페이스 추가).
- Patch: 특정 오브젝트의 특정 속성만 선택적으로 변경 → “수술적인 접근 방식”.

### patch의 주요 작업 유형
- `add`: 속성/항목을 추가 (예: 컨테이너 추가).
- `remove`: 속성/항목을 제거 (예: 레이블 삭제).
- `replace`: 속성 값을 다른 값으로 교체 (예: replicas=1 → replicas=5).

### 패치 정의 시 필요한 요소
1. 작업 유형(op) – add, remove, replace
2. 대상(target) – 어떤 리소스에 적용할지 지정
  - kind, name, namespace, labelSelector, annotationSelector 등으로 일치 조건 설정
3. 경로(path) – YAML 트리에서 속성이 위치한 경로 (예: `/metadata/name`, `/spec/replicas`)
4. 값(value) – 교체하거나 추가할 값 (remove는 필요 없음)

### JSON 6902 패치
- 구성 요소: target + patch 세부정보
- 경로 기반으로 속성을 지정해 수정
- 예시
  - `/metadata/name` → “api-deployment”를 “web-deployment”로 교체
  - `/spec/replicas` → 1을 5로 교체
- RFC 6902 표준 기반

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
        - name: nginx
          image: nginx
```

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patches: |-
      - op: replace
        path: /spec/replicas
        value: 5
```

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patches: |-
      - op: replace
        path: /metadata/name
        value: web-deployment
```

### 전략적 병합 패치(Strategic Merge Patch)
- 일반 Kubernetes YAML과 거의 동일한 형태
- 원본 YAML을 일부만 남겨 원하는 속성만 변경
- 예시

```yaml
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: api-deployment
      spec:
        replicas: 5
```

- 원래 값(1)이 새로운 값(5)로 병합됨
- 장점: 익숙하고 가독성 높음

### 비교 및 활용
- JSON 6902: 세밀한 조작 가능, 표준화된 방식.
- 전략적 병합 패치: 일반 YAML 기반, 직관적이고 가독성 좋음.
- 두 가지 모두 혼합 사용 가능. 취향과 상황에 따라 선택.

## Different Types of Patches
### 패치 정의 방식
- 인라인(In-line) 방식
  - 지금까지 다룬 것처럼 kustomization.yaml 안에서 직접 패치를 정의.
  - 간단한 경우에 적합.
- 별도 파일(File) 방식
  - kustomization.yaml에서는 대상 리소스와 패치 파일 경로만 지정.
  - 실제 패치 내용은 별도의 YAML 파일에 작성.
  - 패치가 많아져 복잡해질 때 유용.

### 두 방식의 공통점
- JSON 6902 패치와 전략적 병합 패치 모두 적용 가능.
- 인라인 정의든 파일 분리든, 기능적으로 동일하게 동작.
- 상황과 복잡도에 따라 선택하면 됨.

## Patches Dictionary
### JSON 6902 패치
- 키 교체(Replace)
  - op: replace
  - path: /spec/template/metadata/labels/component
  - value: web  
    → 기존 component: api → component: web 으로 변경

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: replace
        path: /spec/template/metadata/labels/component
        value: web
```

- 키 추가(Add)
  - op: add
  - path: /spec/template/metadata/labels/org
  - value: kodekloud  
    → component: api 레이블에 org: kodekloud 추가

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: add
        path: /spec/template/metadata/labels/org
        value: kodekloud
```

- 키 제거(Remove)
  - op: remove
  - path: /spec/template/metadata/labels/org  
    → org: kodekloud 레이블 삭제

```yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: remove
        path: /spec/template/metadata/labels/org
```

### 전략적 병합 패치 (Strategic Merge Patch)
- 교체: 원래 Deployment YAML 일부를 복사 → labels 아래 값만 component: web으로 변경.

```yaml
patchesStrategicMerge:
  - label-replace.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        component: web
```

- 추가: 기존 labels에 org: kodekloud 추가.

```yaml
patchesStrategicMerge:
  - label-add.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        org: kodekloud
```

- 삭제: 삭제할 키를 null 값으로 설정 → org: null → 적용 시 레이블이 제거됨.

```yaml
patchesStrategicMerge:
  - label-remove.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        org: null
```

### 차이점 & 활용
- JSON 6902 패치
  - RFC 기반, op/path/value 구조.
  - 세밀하고 선언적인 수정.
- 전략적 병합 패치
  - Kubernetes YAML과 유사해 직관적, 가독성 좋음.
  - 키 제거 시 null로 지정해야 한다는 특징.

## Patches List
### 목록(List)에서의 교체
- JSON 6902
  - 배열의 인덱스를 사용해 특정 항목을 지정 (/spec/template/spec/containers/0/image)
  - op: replace + value 로 변경
  - 예: 첫 번째 컨테이너 nginx → haproxy 로 교체

```yaml
patches:
  - target:
      kind: Deployment
      name: web-deployment
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: haproxy
      - op: replace
        path: /spec/template/spec/containers/0/name
        value: haproxy
```

- Strategic Merge Patch
  - 컨테이너 name 을 기준으로 일치시킨 뒤 image 속성만 새 값으로 지정
  - 예: nginx 컨테이너의 이미지를 haproxy 로 교체

```yaml
patchesStrategicMerge:
  - container-replace.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  template:
    spec:
      containers:
        - name: nginx   # 기존 컨테이너 이름 기준
          image: haproxy
          name: haproxy
```

### 목록에 항목 추가
- JSON 6902
  - op: add 사용
  - path: /spec/template/spec/containers/- → - 는 목록 끝에 추가 의미
  - 값(value)에는 새 컨테이너 정의 (haproxy)

```yaml
patches:
  - target:
      kind: Deployment
      name: web-deployment
    patch: |-
      - op: add
        path: /spec/template/spec/containers/-
        value:
          name: haproxy
          image: haproxy
```

- Strategic Merge Patch
  - 컨테이너 배열 아래에 새 컨테이너 정의만 추가
  - 병합 시 기존 항목과 합쳐져 최종적으로 2개 이상 컨테이너가 됨

```yaml
patchesStrategicMerge:
  - container-add.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  template:
    spec:
      containers:
        - name: haproxy
          image: haproxy
```

### 목록에서 항목 삭제
- JSON 6902
  - op: remove
  - path에 제거할 항목의 인덱스 지정 (/spec/template/spec/containers/1)
  - 값은 필요 없음

```yaml
patches:
  - target:
      kind: Deployment
      name: web-deployment
    patch: |-
      - op: remove
        path: /spec/template/spec/containers/1
```

- Strategic Merge Patch
  - $patch: delete 지시문 사용
  - 삭제하려는 컨테이너를 name 으로 지정 (name: database)
  - 적용 시 해당 컨테이너가 제거됨

```yaml
patchesStrategicMerge:
  - container-remove.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  template:
    spec:
      containers:
        - name: database
          $patch: delete
```

## Overlays
### 오버레이(Overlay) 개념
- Base: 모든 환경에서 공통으로 사용하는 기본 리소스 정의 저장소
- Overlay: 개발(dev), 스테이징(staging), 프로덕션(prod) 등 환경별로 필요한 수정 사항을 정의
- Base의 리소스를 불러와 환경에 맞는 패치를 적용하거나 새 리소스를 추가할 수 있음

### 오버레이 동작 방식
1. 폴더 구조

```
k8s/
  base/
    kustomization.yaml
    nginx-depl.yaml
  overlays/
    dev/
      kustomization.yaml
    staging/
      kustomization.yaml
    prod/
      kustomization.yaml
```

2. 각 Overlay의 kustomization.yaml에는
- `bases` → base 디렉토리 참조
- `patches` → 환경별 패치 정의
- `resources` → 환경별 새 리소스 추가 가능

### 패치 예시 (복제본 변경)
- Base: replicas: 1

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
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
          image: nginx:1.21
---
resources:
  - nginx-depl.yaml
```

- Dev Overlay: 패치로 replicas: 2

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
---
bases:
  - ../../base

patchesStrategicMerge:
  - replicas-patch.yaml
```

- Prod Overlay: 패치로 replicas: 3

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
---
bases:
  - ../../base

patchesStrategicMerge:
  - replicas-patch.yaml

resources:
  - grafana-depl.yaml
```

### 추가 리소스 가능
- Overlay에는 Base에 없는 새로운 리소스도 정의 가능
- 예: Prod 환경에만 grafana-depl.yaml 추가

### 유연한 디렉토리 구조
- Base와 Overlay를 기능별/서비스별 하위 디렉터리로 나눌 수 있음
- 디렉토리 구조는 강제되지 않으며, 필요한 방식으로 자유롭게 구성 가능