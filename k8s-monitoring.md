# k8s Monitoring

| 구분              | DataDog                                        | Prometheus                                                   |
| ----------------- | ---------------------------------------------- | ------------------------------------------------------------ |
| 구축 형태         | SaaS (Cloud)                                   | 직접 구축                                                    |
| 설치              | 간편                                           | 그라파나까지 연계할 경우 상대적으로 복잡                     |
| 시각화            | 강력한 그래프 기능 제공                        | 그래프가 취약하기 때문에 그라파나와 연계 필요                |
| 쿠버네티스 이벤트 | 쿠버네티스 클러스터에서 발생하는 이벤트를 수집 | 쿠버네티스 클러스터에서 발생하는 이벤트를 수집하기 위해 또 다른 솔루션이 필요함 |
| 알람              | 유형별 알람 설정에 제한적                      | Prometheus AlertManager or Grafana Alarm                     |

## 1. Prometheus

![Prometheus-Grafana](https://prometheus.io/assets/architecture.png)

```bash
# k8s-master (root)
# 네임스페이스 생성
$ kubectl create ns monitoring

# API 접근 허용 for 프로메테우스 (클러스터 역할 설정 및 역할 바인딩)
$ vi prometheus-cluster-role.yaml
$ kubectl apply -f prometheus-cluster-role.yaml

# 프로메테우스 수집 메트릭 정의 (메트릭, 주기, 알람)
$ vi prometheus-config-map.yaml
$ kubectl apply -f prometheus-config-map.yaml

# 프로메테우스 파드 생성
$ vi prometheus-deployment.yaml
$ kubectl apply -f prometheus-deployment.yaml

# 노드 메트릭 수집 에이전트 생성
$ vi prometheus-node-exporter
$ kubectl apply -f prometheus-node-exporter.yaml

# 외부에서 프로메테우스 서비스 접속 허용
$ vi prometheus-svc.yaml
$ kubectl apply -f prometheus-svc.yaml

# 파드 조회
$ kubectl get pod -n monitoring
```

- Master Node의 Public IP:30001으로 접속할 수 있도록 방화벽 규칙 생성
- 브라우저에서 `http://35.216.14.122:30001` 주소로 프로메테우스 UI 접속



## 2. Grafana

```bash
# k8s-master (root)
# 그라파나 디플로이먼트 및 서비스 생성
$ vi grafana.yaml
$ kubectl apply -f grafana.yaml
```

- Master Node의 Public IP:30004으로 접속할 수 있도록 방화벽 규칙 생성

- 브라우저에서 `http://35.216.14.122:30004` 주소로 프로메테우스 UI 접속

- 프로메테우스 연동 (Add DataSource with URL)

  ```bash
  # 프로메테우스 서비스 클러스터 IP 조회
  $ kubectl get svc -n monitoring
  ```

  	- Cluster-IP : 10.99.78.18

  	- URL : `http://<Cluster-IP>:8080`



## 3. k8s Resource Management

### 3.1. LimitRange

- 자원의 사용량을 조정 (ex. CPU 사용률이 계속 증가하지 못하도록 설정)
- 1 milicore = CPU의 1/1000 (800 milicore = 0.8 코어)

```bash
# k8s-master (root)
# CPU 사용률 최소/최대 지정
$ vi set-limit-range.yaml
$ kubectl apply -f set-limit-range.yaml
$ kubectl get limitrange
$ kubectl describe limitrange set-limit-range

# CPU 요청량과 최대 사이즈 지정
$ vi pod-with-cpu-range.yaml
$ kubectl create-f pod-with-cpu-range.yaml

$ kubectl get pods
$ kubectl decribe pod pod-with-cpu-range
```



### 3.2. AutoScaling

- 파드의 자원 사용률이 너무 낮거나 높을 때 리소스의 크기를 조정
- 자원 사용량의 제한(LimitRange)은 서비스 속도에 문제가 발생할 수 있으므로 서비스에 어떠한 영향도 주지 않으면서 빠른 속도를 보장하기 위해 사용하는 것이 오토스케일링이다.
- 시스템 스스로 자원 사용률을 감지하여 자동으로 파드를 늘려주거나 줄임
- 메트릭스 서버에 각 파드에서 사용중인 자원(CPU, Mem)이 어느 정도인지 질의하고, HorizontalPodAutoScaler가 계산한 만큼의 파드의 수로 조정

```bash
# k8s-master (root)
# 메트릭스 서버 설치
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 메트릭스 디플로이먼트 설정 변경
$ kubectl edit deployments.apps -n kube-system metrics-server
...
spec:
  containers:
  - args:
    - --cert-dir=/tmp
    - --secure-port=10250
    - --kubelet-insecure-tls=true # Add Here
    - --kubelet-preferred-address-types=InternalIP # Change Here
    # - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-use-node-status-port
    - --metric-resolution=15s
...

# API 서버 설정 변경
$ vi /etc/kubernetes/manifests/kube-apiserver.yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=10.178.0.12
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-aggregator-routing=true # Add here
...

# 오토스케일링 테스트를 위한 디플로이먼트(사용률 제한)와 서비스 생성
$ vi php-apache.yaml
$ kubectl apply -f php-apache.yaml

# 오토스케일링 설정
$ kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
$ kubectl get hpa                      # 부하 확인 (평상시)
$ kubectl top pods
$ kubectl get deployment php-apache    # 파드 수 확인 (평상시)

# 부하 발생기 테스트
$ kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -o- http://php-apache; done"

$ kubectl get hpa                      # 부하 확인 (부하 발생시)
$ kubectl get deployment php-apache    # 파드 수 확인 (부하 발생시)
```