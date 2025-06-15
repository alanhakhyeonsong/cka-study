## Helm Basics
### Kubernetes의 한계와 yaml 관리의 어려움
- 다수의 YAML 파일 존재: Kubernetes 앱을 구성할 때 여러 YAML 파일로 나뉘어 있으며, 변경 시 실수 가능성 존재.
- 삭제의 번거로움: 앱 제거 시 연관된 오브젝트들을 일일이 기억하고 삭제해야 함.
- 하나의 YAML에 모으더라도 한 파일에 전부 작성해도 크기가 커지면 유지보수가 어려움
- Kubernetes는 앱 단위가 아닌 오브젝트 단위로만 인식.
- 배포(Deployment), 서비스(Service), 시크릿(Secret), 볼륨(PV) 등 다양한 구성 요소를 별개로 관리.
- 앱이란 개념이 없어 전체를 묶어서 처리하기 어려움.

### Helm?
- Helm은 Kubernetes의 패키지 관리자로, 하나의 앱을 패키지처럼 관리.
- 하나의 명령으로 수십 개의 오브젝트를 묶어서 설치/제거/업그레이드 가능.
- Helm은 앱의 구성 요소들을 추적하며 릴리즈 단위로 관리함.
- 복잡한 게임도 인스톨러 하나로 설치하는 것처럼, Helm은 앱의 모든 YAML 파일을 하나의 차트(Chart)로 묶어, 단일 명령으로 Kubernetes에 설치.

| 기능       | 설명                              |
| -------- | ------------------------------- |
| 🧱 구조화   | 앱 구성 요소를 패키지(Chart)로 그룹화        |
| ⚙️ 값 파일  | `values.yaml` 파일로 설정값을 한 곳에서 제어 |
| 🔄 업그레이드 | 설정값만 바꾸면 앱 전체를 업그레이드 가능         |
| ↩️ 롤백    | 리비전을 추적하여 쉽게 이전 상태로 되돌릴 수 있음    |
| 🧹 삭제    | 하나의 명령으로 관련 오브젝트 모두 제거 가능       |

Helm은 Kubernetes 앱을 더 이상 오브젝트 모음이 아니라, 하나의 애플리케이션으로 관리할 수 있게 해주는 도구다. YAML 파일의 복잡성과 수작업의 부담을 줄여주며, 패키지 설치기처럼 앱의 생애 주기 전체를 효율적으로 관리할 수 있게 돕는다.

```bash
# helm install [release-name][chart-name]
$ helm install wordpress
$ helm upgrade wordpress
$ helm rollback wordpress
$ helm uninstall wordpress
```

## Helm 2 vs Helm 3
🔀 Helm 2 vs Helm 3의 주요 차이점
- Tiller 제거
  - Helm 2: 클러스터 내부에 Tiller(서버 컴포넌트)를 설치해야 했으며, 보안 위험이 존재 (무제한 권한)
  - Helm 3: Tiller 완전 제거, Helm 클라이언트가 직접 클러스터와 통신
    - 보안성 향상: Kubernetes의 RBAC(Role-Based Access Control)와 자연스럽게 통합
    - 사용자의 kubectl 권한과 동일하게 Helm도 작동
- 리비전 관리 및 롤백
  - Helm은 차트 설치/변경 시마다 **리비전(Snapshot)**을 생성
  - eg. 최초 설치 → 리비전 1, 업그레이드 → 리비전 2, 롤백 → 리비전 3
  - Helm 2: 롤백 시 "이전 차트 vs 현재 차트"만 비교함 → 수동 변경사항은 무시됨
  - Helm 3: 3-way 전략 병합 패치 사용
    - "이전 차트", "현재 차트", "라이브 상태(Kubernetes 실시간 상태)" 3가지를 비교하여
    - 수동 변경도 감지하고, 올바르게 롤백 처리함
    - 업그레이드 시에도 사용자 커스텀 상태를 보존함

| 기능             | 설명                               |
| -------------- | -------------------------------- |
| 🧱 Tiller 제거   | 구조 간소화 및 보안 향상                   |
| 🔐 RBAC 연동     | 사용자의 Kubernetes 권한 그대로 사용        |
| 🧠 3-way 병합 패치 | 수동 변경사항도 감지하여 정교한 롤백 및 업그레이드 수행  |
| 🕰 리비전 관리      | 각 변경마다 스냅샷 생성, 언제든지 이전 상태로 롤백 가능 |
| ⬆️ 업그레이드 보존    | 수동 변경이 삭제되지 않음                   |

- Helm 3는 Helm 2보다 간단하고, 보안성이 뛰어나며, 사용자 경험이 향상된 버전이다.
- Tiller 제거로 구조가 단순해졌고, 3-way 병합 전략 덕분에 수동 변경까지 고려한 정확한 롤백과 업그레이드가 가능해졌다.

## Helm Components
-  Helm CLI (명령줄 유틸리티)
  - 로컬 시스템에 설치하여 차트 설치, 업그레이드, 롤백 등을 수행
  - helm install, helm upgrade, helm rollback 등의 명령 사용
- 차트(Chart)
  - 여러 YAML 파일이 포함된 디렉터리 구조의 패키지
  - Kubernetes 클러스터에서 필요한 오브젝트를 정의
  - 템플릿(.tpl) 파일과 값(values.yaml)을 통해 동적 생성 가능
  - 외부 차트를 기반으로도 애플리케이션 설치 가능
- 릴리스(Release)
  - 하나의 Helm 차트를 사용하여 앱을 설치한 인스턴스
  - 동일한 차트를 여러 번 설치하면 서로 다른 릴리스 생성 가능 (예: 운영용, 개발용)
  - 각 릴리스는 **여러 리비전(Revision)**을 가질 수 있으며, 각 리비전은 설치 시점의 스냅샷

### 리비전과 스냅샷
- 애플리케이션 구성 변경 시 자동으로 새 리비전 생성
  - eg. 이미지 버전 변경, 복제본 수 변경 등
- 각 리비전은 되돌릴 수 있는 스냅샷 역할

### Helm 차트의 소스: Repository
- 차트는 공개 Helm Repository에서 제공됨.
- 대표적인 Repo: Bitnami, TrueCharts, 앱코드, Artifact Hub 등
- https://artifacthub.io 에서 6,300개 이상의 Helm 패키지 검색 가능
- 인증된 퍼블리셔(공식 배지) 차트 사용 권장

| 파일명           | 역할                             |
| ------------- | ------------------------------ |
| `Chart.yaml`  | 차트 메타정보 포함                     |
| `values.yaml` | 사용자 정의 설정값 저장 (주로 수정하는 파일)     |
| `templates/`  | Kubernetes 리소스를 생성하는 템플릿 파일 모음 |

- Hello World 앱에는 Deployment, Service 오브젝트만 존재
- 복잡한 앱(예: WordPress)은 많은 템플릿과 설정이 포함됨

### 릴리스 이름 지정의 이유
- 같은 차트로 여러 릴리스를 독립적으로 관리 가능
- 예: my-site, my-second-site 두 릴리스
- 독립적 수정, 업그레이드, 삭제 가능
- 개발용/운영용 릴리스를 분리하여 안전하게 테스트 가능

### 핵심 요약
- 차트: Helm의 배포 단위로 YAML 템플릿 집합
- 릴리스: 차트를 기준으로 설치된 앱 인스턴스
- 리비전: 릴리스의 상태 변경 이력 (스냅샷처럼 작동)
- `values.yaml`: 사용자 설정의 중심
- Artifact Hub 등에서 차트를 손쉽게 검색하고 설치 가능

## Helm Charts

```yaml
# Service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: hello-world

---
# Deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replica: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: {{ .Value.image.repository }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP

---
# values.yaml
replicaCount: 1

image:
  repository: nginx

---
# Chart.yaml
apiVersion: v2
appVersion: "1.16.0"
name: hello-world
description: A web application

type: application
```

### 차트(Chart)의 구성 요소
```
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```
Helm 차트는 특정 구조와 파일로 구성된 디렉토리다. 주요 구성 요소는 다음과 같다.

1. Chart.yaml (메타데이터 파일)
- API Version: v1 또는 v2 (Helm 3용 차트는 v2 사용)
- name: 차트 이름
- version: 차트 자체의 버전 (앱 버전과 다름)
- appVersion: 배포할 애플리케이션의 버전 (예: WordPress 5.8)
- description: 차트 설명
- type: application(기본) 또는 library
- dependencies: 다른 차트(예: MariaDB 등) 의존성 정의
- keywords, maintainers, home, icon 등 검색과 문서화를 위한 필드 포함

2. values.yaml (설정값 파일)
- 사용자가 설정 가능한 값들을 정의
- 템플릿에서 이 값을 참조하여 실제 YAML 파일이 생성됨
- 가장 자주 수정되는 파일 (예: 이미지 이름, 복제본 수 등)

3. templates/ 디렉터리
- 실제 Kubernetes 리소스(Deployment, Service 등)를 생성하는 템플릿 YAML 파일들
- Go 템플릿 문법을 사용하여 `values.yaml`의 값과 결합됨

4. 기타 파일
- `README.md`: 차트 설명
- `LICENSE`: 라이선스
- `charts/`: 종속 차트들
- `Chart.lock`: 종속성 잠금 파일

### 차트의 작동 방식
- Helm은 차트를 기반으로 최종 결과를 생성하고, 해당 템플릿과 값을 조합해 실제 Kubernetes 매니페스트를 만들어 클러스터에 적용.
- 사용자는 목적만 명시하면 Helm이 나머지 실행 경로를 자동으로 계산.

### Helm 3의 특징 (vs Helm 2)
- `Chart.yaml`에 API Version 필드(v2) 추가됨 (차트의 버전을 명확하게 구분)
- dependencies 필드와 type 필드는 Helm 3에서 새로 도입된 기능
- Helm 2로 작성된 차트는 apiVersion: v1, Helm 3용 차트는 apiVersion: v2 권장

### 예시: WordPress + MariaDB 차트
- WordPress는 프론트엔드 앱, MariaDB는 백엔드 DB로 구성된 2계층 애플리케이션
- MariaDB는 독립 차트로 되어 있고, 종속성으로 연결만 하면 Helm이 알아서 설치
- 두 차트를 병합할 필요 없음