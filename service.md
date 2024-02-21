# Service

## 1. Service

쿠버네티스 파드는 클러스터 내의 노드들로 옮겨 다닌다. 이 때 파드의 IP가 새로운 IP로 변경되기 때문에 내/외부에서 접근하기 어렵다는 문제가 있다. 이러한 문제를 해결하기 위해 사용하는 것이 서비스이다. 서비스를 사용하면 파드가 클러스터 내 어디에 있든지 마치 고정된 IP처럼 접근할 수 있다.

### Service Type

| 클러스터 내부 접속 용도 | 클러스터 외부 접속 용도                                   |
| ----------------------- | --------------------------------------------------------- |
| ClusterIP               | ExternalName<br />NodePort<br />LoadBalancer<br />Ingress |



## 1.1. ClusterIP

![img](https://blog.kakaocdn.net/dn/88dPH/btq5Fdmb6Aj/59Ka0q7glHTKSYKGeu1nmk/img.png)

- 클러스터 내 파드에 접근할 수 있는 가상의 IP(Default)이다.
- ClusterIP는 클러스터 내부에서만 사용할 수 있으며, 외부에서는 파드에 접속할 수 없다.
- 서비스 생성시 ClusterIP를 별도로 지정하지 않으면(None) ClusterIP가 없는 **Headless Service**가 생성된다.
  - 로드 밸런싱이나 서비스 IP가 필요 없을 때 Headless Service를 사용한다.
  - Headless Service에 셀렉터를 설정하면 API를 통해서 접근할 수 있는 엔드포인트가 만들어지지만, 셀렉터가 없으면 엔드포인트가 만들어지지 않는다. 

```yaml
# ClusterIP.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusterip-nginx
spec:
  selector:
    matchLabels:
      run: clusterip-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: clusterip-nginx
    spec:
      containers:
      - name: clusterip-nginx
        image: nginx
        ports:
        - containerPort: 80
```

```bash
# Deployment(Pod) 생성
$ kubectl apply -f ClusterIP.yaml
$ kubectl get pods -l run=clusterip-nginx -o wide

# 서비스 생성 (manifest/expose)
$ kubectl expose deployment/clusterip-nginx
$ kubectl get svc clusterip-nginx
$ kubectl describe svc clusterip-nginx # ClusterIP : 10.105.167.130
$ kubectl get ep clusterip-nginx # Pod의 엔드포인트 : 10.244.2.32:80, 10.244.3.67
```

### Endpoint

- 쿠버네티스에서 서비스와 파드의 통신은 **서비스의 셀렉터(selector)와 파드의 레이블(labels)**을 이용한다.

- 파드의 레이블이 서비스의 셀렉터에서 사용한 이름과 같으면 서비스는 해당 파드로 트래픽을 보낸다.
- 서비스가 엔드포인트와 파드들을 매핑하여 관리하기 때문이다.

```bash
# ClusterIP를 이용하여 콘텐트 다운로드
$ kubectl run busybox --rm -it --image=busybox /bin/sh
$ wget 10.105.167.130 # ClusterIP
$ cat index.html
$ exit

# 오브젝트 생성 명령어(create, apply, run)
# kubectl run/expose : 생성형
# kubectl create : 
```

- 오브젝트 생성 명령어 (create,apply,run)
  - `run`, `expose`(생성형) :  컨테이너가 원하는 대로 작동하는지 빠르게 확인하기 위한 명령. 오브젝트 생성 및 변경에 대한 히스토리를 저장하지 않기 때문에 변경이력 관리가 어렵다. 주로 개발환경에서 사용된다.
  - `create`(명령형) : 최소 하나의 yaml 또는 json 파일을 이용해서 오브젝트를 생성해야 하며 생성된 파일을 통해 히스토리를 추적한다.
  -  `apply`(선언형) : 애너테이션에 정보가 저장되어 자동으로 히스토리를 관리할 수 있다. 따라서 선언형에서는 create/replace 명령을 지원하지 않는다.

### Annotation

- 오브젝트에 메타데이터를 할당할 수 있는 **주석**같은 개념이다.
- 키-값 구조를 가지며 레이블이 레이블 셀렉터를 이용해 검색과 식별을 할 수 있지만 애너테이션에서는 입력은 가능해도 검색은 되지 않는다.
- 쿠버네티스 클러스터 API 서버가 애너테이션에 지정된 메타데이터를 참조해 동작한다.



## 1.2. ExternalName

<img src="https://blog.kakaocdn.net/dn/cfJF5S/btrBIg669fU/Slxn8QKN7ecA1WXUOk0s60/img.png" alt="img" style="zoom:50%;" />

- 클러스터 내부에서 외부의 엔드포인트에 접속하기 위한 서비스이다. 이 때 엔드포인트는 클러스터 외부에 위치하는 데이터베이스나 API 등을 의미한다.
- 프록시나 특정 이름(주소)을 다른 이름으로 자동으로 변환하는 리다이렉트 용도로 사용한다.

```yaml
# svc-ExternalName.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: myservice.test.com # external-service와 연결하려는 외부 도메인(FQDN) 지정
```



## 1.3. NodePort

![img](https://blog.kakaocdn.net/dn/EIArC/btq5u3ZGQt4/SJHUeDKwJVkaEorWjWvO2k/img.png)

- **모든 워커 노드에 특정 포트(노드포트)를 열고** 여기로 들어오는 모든 요청을 노드포트 서비스로 전달한다.
- 노드포트 서비스는 해당 업무를 처리할 수 있는 파드로 요청을 전달한다.

- 노드포트는 30000~32767로 제한되어 있다.
- 포트당 하나의 서비스만 사용할 수 있다.
- 노드나 가상 머신의 IP 주소가 변경되면 반드시 반영해야 한다.

```yaml
# nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 2
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

```yaml
# nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 31472
    targetPort: 80
  selector:
    app: nginx
```

<img src="https://github.com/davidhyun/k8s-practice/assets/64063767/1f51326f-9283-4a9b-9fbf-baf5d3c6a110" alt="image" style="zoom:50%;" />

```bash
$ kubectl apply -f nginx-deploy.yaml
$ kubectl get deployments
$ kubectl apply -f nginx-svc.yaml
$ kubectl get svc | grep nginx-svc
$ curl 10.178.0.13:31472 # worker1-IP:NodePort
$ curl 10.178.0.10:31472 # worker2-IP:NodePort
$ curl 10.178.0.11:31472 # worker3-IP:NodePort
```



## 1.4. LoadBalancer

**부하를 분산**하기 위해 사용하는 것이 로드밸런서이다. 쿠버네티스에서 L7 로드밸런서의 기능은 Ingress를 사용하여 구현한다.

<img src="https://www.densify.com/wp-content/uploads/article-k8s-capacity-kubernetes-service-overview.svg" alt="Overview of Kubernetes Service" style="zoom:50%;" />

| 구분            | 설명                                                         | OSI 7계층                                 |
| --------------- | ------------------------------------------------------------ | ----------------------------------------- |
| L4 LoadBalancer | **IP** 또는 **Port**로 트래픽을 분산                         | 네트워크 계층(IP)<br />전송 계층(TCP/UDP) |
| L7 LoadBalancer | **URL, HTTP 헤더, 쿠키** 등의 사용자의 요청을 기준으로 트래픽을 분산 | 어플리케이션 계층(HTTP,FTP,SMTP)          |

```yaml
# httpd-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: httpd-service
spec:
  selector:
    app: httpd
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  externalIPs:
  - 10.178.0.13 # 워커노드1의 내부IP
```

```bash
$ kubectl create deployment httpd --image=httpd
$ kubectl create -f httpd-service.yaml
$ kubectl get svc
$ curl -i 10.178.0.13
```



```yaml
# service/load-balancer.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: load-balancer
  name: hello-world
spec:
  replicas: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: load-balancer
  template:
    metadata:
      labels:
        app.kubernetes.io/name: load-balancer
    spec:
      containers:
      - image: gcr.io/google-samples/node-hello:1.0
        name: hello-world
        ports:
        - containerPort: 8080
```

```bash
$ kubectl apply -f load-balancer.yaml
$ kubectl get deployments hello-world
$ kubectl describe deployments hello-world
$ kubectl get replicasets
$ kubectl describe replicasets

# 디플로이먼트 외부 노출
$ kubectl expose deployment hello-world --type=LoadBalancer --name=exservice
$ kubectl get services exservice # 8080:31281 (외부 접속시 31821포트로 접속)
$ kubectl describe services exservice # NodePort: 31821/TCP
$ kubectl get pods --output=wide

$ curl http://10.178.0.13:31821 # worker1 내부IP: 10.178.0.13
$ curl http://10.178.0.10:31821 # worker2 내부IP: 10.178.0.10
$ curl http://10.178.0.11:31821 # worker3 내부IP: 10.178.0.11
```



## 1.5. Ingress

- 클러스터 외부에서 내부로 접근하는 요청(HTTP, HTTPS)을 어떻게 처리할지 정의해둔 규칙들의 모음이다.
- 어떤 URL 경로로 요청이 왔을 때 어떤 서비스로 연결하라는 규칙을 정의한 것.

<img src="https://miro.medium.com/v2/resize:fit:1200/1*KIVa4hUVZxg-8Ncabo8pdg.png" alt="Kubernetes NodePort vs LoadBalancer vs Ingress? When should I use what? |  by Sandeep Dinesh | Google Cloud - Community | Medium" style="zoom:50%;" />

```bash
# ingress-nginx-controller.yaml
$ kubectl apply -f ingress-nginx-controller.yaml
$ kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
```

- ValidatingWebhookConfiguration은 파드 생성시 사용자가 정의한 자원(manifest.spec)에 문제가 없는지 검증하기 위해 사용한다.
- 하지만 ValidatingWebhookConfiguration을 활성 상태로 두면 인그레스 테스트 과정에서 오류가 발생하니 삭제한다.

```yaml
# cafe.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee
spec:
  replicas: 2
  selector:
    matchLabels:
      app: coffee # This
  template:
    metadata:
      labels:
        app: coffee
    spec:
      containers:
      - name: coffee
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: coffee-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: coffee # coffee service > coffee pod
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tea
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tea # This
  template:
    metadata:
      labels:
        app: tea
    spec:
      containers:
      - name: tea
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tea-svc
  labels:
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: tea # tea service > tea pod
```

```bash
$ kubectl apply -f cafe.yaml
$ kubectl get deploy,svc,pods
```

```yaml
# cafe-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cafe-ingress
  annotations:
    # kubernetes.io/ingress.class: nginx # deprecated
    spec.ingressClassName: nginx
spec:
  rules:
  - http:
      paths:
      - path: /tea # /tea URL 요청시 tea-svc로 연결
        pathType: Prefix
        backend:
          service:
            name: tea-svc
            port:
              number: 80
      - path: /coffee # /coffee URL 요청시 coffee-svc로 연결
        pathType: Prefix
        backend:
          service:
            name: coffee-svc
            port:
              number: 80
```

```bash
$ kubectl apply -f cafe-ingress-yaml
$ kubectl get ingress
$ kubectl get pod -n ingress-nginx # ingress-nginx-controller 파드 상태가 Running이면 정상

# [CrashLoopBackOff] 파드가 시작과 비정상 종료를 반복하는 경우 컨트롤러 삭제 후 다시 실행
$ kubectl delete -f ingress-nginx-controller.yaml
$ kubectl apply -f ingress-nginx-controller.yaml

$ kubectl get svc -n ingress-nginx # 80:31064/TCP,443:31579/TCP (외부 접속시 31064포트로 접속)

# 같은 주소로 접속했지만 응답 파드가 다를 수 있음 (클라우드 상에서는 확인 안됨, 404)
$ curl -i http://10.178.0.11:31064/coffee # 200, Server Name: coffee-1
$ curl -i http://10.178.0.11:31064/coffee # 200, Server Name: coffee-2
```