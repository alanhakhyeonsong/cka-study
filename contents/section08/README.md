# Storage
## Storage in Docker
쿠버네티스의 스토리지를 이해하려면, 먼저 Docker에서 스토리지가 어떻게 작동하는지 숙지해야 함  
Docker 스토리지의 두 축
- 저장소 드라이버(Storage Driver)
- 볼륨 드라이버(Volume Driver) & 플러그인

### Docker 스토리지 기본 위치
- 기본 데이터 경로: `/var/lib/docker/` 하위에 이미지, 컨테이너, 볼륨 등 모든 Docker 데이터가 저장됨

### 이미지의 계층화 구조 (Layered Architecture)
- 레이어 생성
	-	Dockerfile 각 명령(RUN, COPY 등)이 새 레이어를 만듦
	-	이전 레이어에서 변경된 파일만 저장
- 빌드 캐시 재사용
	-	동일 기반(예: Ubuntu 베이스, 의존성 설치) 레이어가 있으면 재사용
	-	변경된 부분(소스 코드·진입점)만 다시 빌드 → 시간·디스크 절약

```Dockerfile
FROM ubuntu

RUN apt-get update && apt-get -y install python

RUN pip install flask flask-mysql

COPY . /opt/source-code

ENTRYPOINT FLASK_APP=/opt/source-code/app.py flask run
```

## 컨테이너의 쓰기 가능한 레이어
- 쓰기 가능한 최상위 레이어
- 컨테이너 실행 시 이미지 레이어 위에 생성
- 로그 파일·임시 파일·사용자 수정 파일 등 동적 데이터 저장
- Copy-on-Write
- 이미지 레이어는 불변(읽기 전용)
- 수정 시 해당 파일을 쓰기 레이어로 복사 후 변경

### 데이터 영구화: 볼륨 vs 바인드 마운트
- 볼륨 마운트 (`docker volume create + -v volume_name:/container/path`)
	- `/var/lib/docker/volumes/` 아래에 관리
	- 컨테이너 삭제 시에도 데이터 유지
- 바인드 마운트 (`-v /host/path:/container/path` 또는 `--mount type=bind,source=/host/path,target=/container/path`)
	- 호스트 파일 시스템의 특정 디렉터리를 직접 마운트
	- 외부 저장소나 지정된 호스트 경로를 컨테이너에 연결
- 권장 표기법:
	-	구버전 `-v` 대신 `--mount` 옵션(delta key=value 형식)이 가독성·유연성에서 선호됨

### 스토리지 드라이버(Storage Driver)
- 역할: 이미지 레이어·쓰기 레이어 관리, Copy-on-Write 구현
- 주요 드라이버: AUFS, Overlay2, Device Mapper, Btrfs, ZFS 등
- 선택 기준: 운영체제별 지원 여부·성능·안정성
- 예) Ubuntu 기본은 AUFS, CentOS/Fedora는 Device Mapper 등

## Container Storage Interface
### 확장성을 위한 표준 인터페이스
- 과거 Kubernetes는 Docker 런타임에 종속
- CRI(Container Runtime Interface), CNI(Container Networking Interface)에 이어
- CSI(Container Storage Interface) 도입
- 다양한 스토리지 솔루션을 플러그인 형태로 지원 가능

### CSI의 역할과 구조
-	목적: 컨테이너 오케스트레이터(Kubernetes 등)와 외부 스토리지 시스템 간 통신 표준 정의
-	RPC 호출 방식
  -	CreateVolume: 볼륨 생성 요청
  -	DeleteVolume: 볼륨 삭제 요청
  -	그 외 매개변수 및 에러 코드 교환 규격

### 생태계 및 구현체
- 주요 벤더별 CSI 드라이버 존재:
  - AWS EBS, Azure Disk
  - Dell EMC Isilon, PowerMax, Unity
  - Pure Storage, Hitachi, NetApp, Nutanix, HPE 등
- Kubernetes 외에도 Cloud Foundry, Mesos 등 CNCF 프로젝트들이 CSI 지원

### 표준화와 참고 자료
- CSI 사양(spec)은 GitHub에 공개되어 있으며, 세부 RPC 정의와 인터페이스 규약 확인 가능
- 컨테이너 오케스트레이터는 CSI 드라이버만 구현하면 스토리지 공급업체와 별도 연동 작업 불필요

## Volumes
### 컨테이너/Pod의 일시성
- Docker 컨테이너, Kubernetes Pod는 에페메럴(ephemeral)하여 종료되면 내부 데이터도 함께 사라짐

### 데이터 영구 보존을 위한 볼륨 사용
-	컨테이너(Pod)에 볼륨을 붙여주면 생성된 데이터는 볼륨에 저장되어 컨테이너(Pod) 삭제 후에도 남아 있음

### 단일 노드 예시: HostPath 볼륨
- 볼륨 생성: Kubernetes 볼륨 정의에서 hostPath로 호스트의 특정 디렉터리(`/data`) 지정
- 볼륨 마운트: Pod 스펙에서 volumeMounts를 통해 컨테이너 내부의 경로(`/opt`)에 연결
- 동작: 컨테이너가 생성한 파일이 호스트의 `/data` 디렉터리에 저장 → Pod 삭제 후에도 보존

### HostPath 볼륨의 한계
- 단일 노드에선 동작하나, 다중 노드 클러스터에서는 비추천
- 모든 노드의 로컬 경로가 동일하다는 보장이 없기 때문

### 다양한 스토리지 백엔드 지원
Kubernetes는 여러 볼륨 플러그인을 통해 멀티 노드·영구 스토리지를 지원
- 네트워크 파일시스템: NFS, Ceph FS, Fibre Channel 등
- 클라우드 블록 스토리지
  - AWS EBS
  - Azure Disk
  - GCP Persistent Disk
- 기타 솔루션: Flocker, ScaleIO 등

### 클라우드 예시 (AWS EBS)
- `hostPath` 대신 `awsElasticBlockStore` 필드 사용
- 필수 매개변수: 볼륨 ID, 파일시스템 타입 등 지정

## Persistence Volumes
### 문제 제기
- 기존 방식: Pod 정의마다 직접 volume 설정 → Pod 수가 많아질수록 반복적·수동적 관리 필요
- 단점: 저장소 솔루션 변경 시 모든 Pod 정의를 일일이 수정해야 함

### 중앙 관리형 스토리지 풀
- PersistentVolume (PV)
- 클러스터 관리자(Admin)가 “스토리지 풀”로 미리 생성·관리
- 다양한 백엔드(NFS, AWS EBS, Azure Disk 등)로 구현 가능
- PersistentVolumeClaim (PVC)
- 애플리케이션 사용자가 PV 풀에서 원하는 용량·액세스 모드에 맞춰 “청구”

### PV 리소스 생성 예시
1. 템플릿 작성

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  capacity:
    storage: 1Gi               # 예약할 스토리지 크기
  accessModes:                 # 마운트 방식 지정
    - ReadWriteOnce            # (예: ReadOnlyMany, ReadWriteMany 등)
  hostPath:                    # 로컬 노드 디렉터리 (테스트용)
    path: /data
```

2. 호스트 경로 대체  
프로덕션 환경: `hostPath` 대신 `awsElasticBlockStore` 등 실제 스토리지 필드로 교체

3. 커맨드 실행

```bash
kubectl create -f pv-vol1.yaml    # PV 생성
kubectl get pv                     # 생성된 PV 확인
```

## Persistence Volume Claims
### PV vs PVC 역할 분리
- PersistentVolume (PV)
  - 클러스터 관리자(Admin)가 미리 생성·관리하는 스토리지 풀(클러스터 범위)
- PersistentVolumeClaim (PVC)
  - 애플리케이션 사용자가 “스토리지 요청”을 위해 생성하는 리소스(네임스페이스 범위)

### PVC 바인딩 과정
1. 사용자가 PVC 생성
2. Kubernetes가 PVC의 요구사항
- 요청 용량(storage request)
- 액세스 모드(accessModes)
- 볼륨 모드(volumeMode), 스토리지 클래스(storageClass) 등
3. 일치하는 PV 검색 → 단일 PVC ↔ 단일 PV 일대일 바인딩
- 여러 PV가 매칭될 경우 라벨·선택기(selector)로 특정 PV 지정 가능
- 매칭 PV 없으면 PVC는 Pending 상태로 대기
4. 새로운 PV가 추가되면 자동 바인딩

### PVC 생성 예시
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce       # 읽기/쓰기 모드 예시
  resources:
    requests:
      storage: 500Mi      # 요청 용량
```

- `kubectl create -f pvc.yaml`으로 생성
- `kubectl get pvc`로 상태 확인 → 바인딩 여부 확인
- `kubectl get pv`로 어느 PV에 묶였는지 확인

4. PVC 삭제 시 PV 처리 정책(Reclaim Policy)
- Retain (기본)
  - PVC 삭제 후에도 PV 유지 → 관리자 수동 삭제·관리
- Delete
  - PVC 삭제 즉시 PV(실제 백엔드 볼륨)도 자동 삭제
- Recycle
  - PVC 삭제 시 PV 데이터만 삭제한 뒤 재사용 가능

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

위 예시는 Deployment, ReplicaSets의 경우도 동일함.

## Storage Class
### 정적 프로비저닝의 한계
- PVC를 사용하려면 사전에 클라우드(예: GCP)에서 디스크를 직접 생성하고
- 그 이름으로 PV를 수동 작성해야 함 → 반복적·수작업 발생

### 동적 프로비저닝 개념
- StorageClass를 통해 프로비저너(provisioner)를 정의
- PVC에 해당 StorageClass를 지정하면
	1. 쿠버네티스가 프로비저너를 호출해 클라우드에 디스크 생성
	2. 자동으로 PV 객체를 만들어 PVC에 바인딩

### StorageClass 설정 예시
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd     # GCE Persistent Disk 프로비저너
parameters:
  type: pd-ssd                        # 디스크 유형 (표준/SSD 등)
  replication-type: regional          # 복제 모드 등
```

### 다양한 프로비저너 및 매개변수
- GCE PD 외에도 AWS EBS, Azure Disk/File, CephFS, Portworx, ScaleIO 등 지원
- 각 프로비저너별로 디스크 유형, IOPS, 복제 정책 등 세부 파라미터 설정 가능

### 스토리지 클래스 활용
- 여러 StorageClass(예: `silver/gold/platinum`)를 미리 정의하여
- 애플리케이션별 요구 성능·내구성에 맞춰 PVC에서 간단히 선택