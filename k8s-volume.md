# k8s volume

## 1. Volume

데이터 저장소. 쿠버네티스에서는 파드가 종료되어도 데이터를 보존할 수 있는 볼륨이라는 저장소를 이용한다.

### 1.1. Volume 유형

| Type      | Location       | Description                                                  | Example                                           |
| --------- | -------------- | ------------------------------------------------------------ | ------------------------------------------------- |
| 임시 볼륨 | 파드 내부      | 파드가 종료될 때 데이터도 같이 삭제                          | emptyDir                                          |
| 로컬 볼륨 | 워커 노드 내부 | 파드가 종료되어도 데이터는 유지되지만, 노드가 종료되면 데이터도 삭제 | hostPath                                          |
| 외부 볼륨 | 노드 외부      | 파드, 노드의 종료와 무관하게 데이터는 항상 보존              | NFS, cephFS, glusterFS, iSCSI, AWS EBS, azureDisk |

#### emptyDir

```yaml
# emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydata # emptydata라는 이름의 파드 생성
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-storage
      mountPath: /data/shared # 해당 경로에 볼륨을 마운트
  volumes:
  - name: shared-storage
    emptyDir: {} # shared-storage 이름으로 emptyDir 기반의 볼륨 생성
```

```bash
$ kubectl apply -f emptydir.yaml
$ kubectl get pod -o wide
$ kubectl exec -it emptydata -- /bin/bash
$ cd /data/shared
```

#### hostPath

- 다른 노드의 파드에서는 볼륨이 생성된 노드의 hostPath 볼륨을 사용할 수 없음
- 리눅스 노드끼리 NFS mount 되어있는 경우에 볼륨 마운트 가능하지만 쿠버네티스 관리 범위 밖임 (비권장)

```yaml
# hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath # hostpath라는 이름의 파드 생성
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: localpath
      mountPath: /data/shared
  volumes:
  - name: localpath
    hostPath:
      path: /tmp # 워커 노드의 /tmp 디렉토리를 hostPath로 사용
      type: Directory
```

```bash
$ kubectl apply -f hostpath.yaml
$ kubectl get pod -o wide
$ kubectl exec -it hostpath -- /bin/bash
$ cd /data/shared
```



## 2. PV & PVC

**영구 볼륨(Persistent Volume, PV)**은 컨테이너, 파드, 노드의 종료와 무관하게 데이터를 영구적으로 보관할 수 있는 볼륨.

디플로이먼트로 파드를 생성할 때 **영구 볼륨을 요청하는 영구 볼륨 클레임(Persistent Volume Claim, PVC)**을 지정한다. 볼륨 클레임에서 볼륨을 할당한다.

![Storage options for applications in an Azure Kubernetes Services (AKS) cluster](https://learn.microsoft.com/ko-kr/azure/aks/media/concepts-storage/aks-storage-options.png)

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual   # 영구 볼륨 클레임의 요청을 해당 영구 볼륨에 바인딩
  capacity:
    storage: 20Gi	# 영구 볼륨으로 20Gi 할당
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"    # 외부 저장소가 준비되지 않았기 때문에 노드의 /mnt/data 디렉터리를 볼륨으로 사용
```

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

```yaml
# pvc-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql    # mysql이라는 디플로이먼트 생성
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8.0.29     # mysql 8.0.29 버전의 파드 생성
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password      # mysql에 접속하기 위한 패스워드
        ports:
        - containerPort: 3306   # mysql에서 사용하는 포트
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: var/mysql   # /var/mysql 디렉터리로 볼륨 마운트
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:       # 영구 볼륨 할당 요청
          claimName: mysql-pv-claim
```

```yaml
# mysql_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  ports:
  - port: 3306
  selector:
    app: mysql
```

```bash
$ kubectl apply -f pv.yaml
$ kubectl apply -f pvc.yaml
$ kubectl apply -f pvc-deployment.yaml
$ kubectl create -f mysql_service.yaml

$ kubectl describe deployment mysql
$ kubectl get pod -o wide
$ kubectl get service
$ kubectl get pv
$ kubectl describe pv mysql-pv-volume
$ kubectl get pvc
$ kubectl describe pvc mysql-pv-claim
```

### PV STATUS

- `Available` : 아직 클레임에 바인딩되지 않은 상태
- `Bound` : 볼륨이 클레임에 의해 바인딩된 상태
- `Released` : 클레임이 삭제되었지만 클러스터에서 아직 리소스가 반환되지 않은 상태
- `Failed` : 볼륨이 자동 반환에 실패한 상태

### PV RECLAIM POLICY

- `Retain` : 영구 볼륨 클레임이 삭제되어도 저장소에 저장되어 있던 파일을 삭제하지 않는 정책
- `Delete` : 영구 볼륨 클레임이 삭제되면 영구 볼륨과 연결된 저장소 자체를 삭제
- `Recycle` : 영구 볼륨 클레임이 삭제되면 영구 볼륨과 연결된 저장소 데이터는 삭제한다. 하지만 저장소 볼륨 자체를 삭제하지는 않는다.

### PVC ACCESS MODES

- `RWO(ReadWriteOnce)` : 하나의 노드에서 해당 볼륨이 읽기/쓰기로 마운트된다.
- `ROX(ReadOnlyMany)` : 볼륨은 많은 노드에서 읽기 전용으로 마운트된다.
- `RWX(ReadWriteMany)` : 볼륨은 많은 노드에서 읽기/쓰기로 마운트 된다.
- `RWOP(ReadWriteOncePod)` : 볼륨이 단일 파드에서 읽기/쓰기로 마운트 된다. 즉, 전체 클러스터에서 단 하나의 파드만 해당 영구 볼륨 클레임을 읽거나 쓸 수 있어야하는 경우 RWOP 모드를 사용한다.