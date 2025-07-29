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

