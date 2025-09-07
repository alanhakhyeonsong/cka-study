# Troubleshooting
## Application Failure
### 애플리케이션 구조
- 웹 Pod: 웹 서버 실행, 웹 서비스로 사용자에게 노출
- DB Pod: 데이터베이스 실행, DB 서비스로 웹 서버에 제공

### 문제 해결 절차
1. 사용자 관점에서 접근 확인
- NodePort IP + curl 등으로 프론트엔드 웹 접근 가능한지 확인
2. 서비스 확인
- 서비스의 selector와 Pod의 label이 일치하는지 확인
- 엔드포인트가 올바르게 잡혔는지 확인
3. Pod 상태 확인
- Pod 상태, 재시작 횟수 확인
- 이벤트(kubectl describe pod)를 확인해 문제 단서 확보
4. 로그 분석
- kubectl logs 로 애플리케이션 로그 확인
- 컨테이너가 재시작 중이라면 --previous 옵션으로 이전 로그 확인
- -f 옵션으로 실시간 로그 모니터링
5. DB 계층 점검
- DB 서비스 접근 가능 여부 확인
- DB Pod 상태와 로그 확인 → 오류 메시지 탐색

### 추가 팁
- 문제 발생 시 **아키텍처 지도(구성도)**를 미리 두고 점검하면 빠르게 원인 추적 가능
- **문제의 양 끝(사용자 ↔ DB)**에서 출발해 중간 레이어를 하나씩 확인
- Kubernetes 공식 문서에 다양한 트러블슈팅 팁이 있음 → 시험 대비에 유용

## Control Plane Failure
### 초기 확인
- Node 상태 점검 → 클러스터 내 모든 노드가 정상인지 확인
- Pod 상태 확인 → Control Plane 관련 Pod가 정상 실행 중인지 확인

### 구성 요소 위치별 확인
- kubeadm 기반 클러스터 → kube-system 네임스페이스에 Control Plane Pod 확인
- 서비스로 배포된 경우 → 마스터 노드에서 kube-apiserver, controller-manager, scheduler 같은 서비스 확인
- 워커 노드 → kubelet, kube-proxy 서비스 상태 확인

### 로그 점검
- kubeadm → kubectl logs 로 Control Plane Pod 로그 확인
- 서비스 기반 배포 → 호스트 OS 로깅 솔루션 이용
- 예: journalctl -u kube-apiserver 로 API 서버 로그 확인

## Worker Node Failure
