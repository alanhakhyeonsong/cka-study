# Logging & Monitoring
## Monitor Cluster Components
- Node 수준 지표
  - Node 수
  - 정상/비정상 Node 개수
  - CPU, Memory, Disk, Network
- Pod 수준 지표
  - Pod 수
  - Pod별 CPU/Memory 사용량

Kubernetes는 기본적으로 풀 기능 모니터링 솔루션이 없다.
- 오픈소스 솔루션
  - Metric Server
  - Prometheus
  - ELK/Elastic Stack
- 상용 솔루션
  - Datadog
  - Dynatrace

### Metric Server
- 클러스터당 1개 배포
- 각 Node/Pod의 메트릭을 수집
- 메모리 내 저장(디스크 저장 X → 히스토리성 데이터 불가)
- 실시간/근접한 시점의 자원 사용량 파악 용도
- 장기 저장 및 분석은 Prometheus 같은 고급 솔루션 필요

### 데이터 수집 경로
- 각 Node → kubelet → cAdvisor 포함
- cAdvisor가 Container/Pod 메트릭 수집
- kubelet API → Metric Server로 노출

### 메트릭 확인 명령어
- Node : `kubectl top nodes`
- Pod : `kubectl top pods`

## Managing Application Logs
### Docker에서 로그 보기
- 컨테이너 예: event-simulator
  - 난수 이벤트 생성 + 웹 서버 시뮬레이션
- 분리(detached) 모드 실행 시 표준 출력이 보이지 않음
- 로그 확인
  - `docker logs <컨테이너ID>`
  - `docker logs -f <컨테이너ID>` (실시간 스트리밍)

### k8s에서 동일 이미지로 Pod 생성
- Pod 정의 파일을 작성해 배포
- 로그 확인
  - `kubectl logs <pod이름>`
  - `kubectl logs -f <pod이름>` (실시간)

### Pod 내부의 여러 컨테이너 로그 보기
- Kubernetes Pod는 여러 컨테이너를 포함 가능
- 예제: event-simulator + image-processor 컨테이너
- 로그 명령
  - `kubectl logs <pod이름> -c <컨테이너이름>`
  - 컨테이너 이름을 명시해야 함
  - 미지정 시 오류 메시지와 함께 이름 명시 요구

요약
- 쿠버네티스의 `kubectl logs` 명령은 Docker의 `docker logs`와 매우 유사
- 단일 컨테이너 Pod는 단순히 Pod 이름만 지정
- 다중 컨테이너 Pod는 -c 옵션으로 컨테이너 이름을 지정