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
## Kustomize ApiVersion & Kind
