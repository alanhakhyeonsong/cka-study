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