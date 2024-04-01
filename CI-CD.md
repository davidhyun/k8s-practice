# CI/CD

![CICD](https://blog.kakaocdn.net/dn/7KhWG/btqXXUv31Zz/kGdisGuYjtKKTliIZMh59K/img.png)

## 1. CI (Continuous Integration)

**지속적 통합**을 뜻하며, 새로운 기능을 만들거나 수정한 소스 코드를 테스트하고 깃허브 같은 저장소에 배포하는 과정을 의미한다. 깃허브에 소스 코드를 저장한 후 수정하면 자동으로 빌드된다.

![Industry Used Cases of Jenkins, and how it Works !!! | by Akhilesh Jain |  Medium](https://miro.medium.com/v2/resize:fit:951/0*IK5Vvf-XK2yUd4j8.jpeg)

### 1.1. Install Jenkins

```bash
# Prerequirement (openjdk-17)
$ sudo apt update
$ sudo apt install -y openjdk-17-jre-headless
$ java --version

# Install Jenkins
$ sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

$ echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install jenkins
$ sudo systemctl status jenkins # enabled 확인 (8080 port)
```

### 1.2. Jenkins Settings

1. 방화벽 규칙 생성 : 8080 port allowed from AP IP
2. k8s-master 외부 IP 브라우저 접속 (http://35.216.14.122:8080)
3. `/var/lib/jenkins/secrets/initialAdminPassword` 조회
4. 기본 플러그인 설치
5. 사용자 계정 설정 (admin/admin)
6. GitHub 레포지토리 액세스 토큰 생성 (권한 설정)

<img width="783" alt="image" src="https://github.com/davidhyun/k8s-practice/assets/64063767/89e4a1f3-c991-4259-881f-f3bbf6086998">

7. Jenkins System Configuration > GitHub 연동 with Token
   (Kind를 `Secret Text`로 하고 Token 입력)

8. Jenkins 아이템 추가

   1. 빌드 방법 지정 : `Github hook trigger for GITScm polling`
   2. Item Configure > GitHub Repository 등록 with Token
      (Kind를 `Username with password`로 하고 password에 Token 입력)
   3. 브랜치 (`*/main`)

9. 빌드 도구 설치 (Maven/Ant/Gradle)

   ```bash
   # k8s-master
   $ wget https://services.gradle.org/distributions/gradle-7.4.2-bin.zip -P /tmp
   
   $ sudo unzip -d /opt/gradle /tmp/gradle-*.zip
   $ ls /opt/gradle/gradle-7.4.2
   $ sudo vi /etc/profile.d/gradle.sh # gradle.sh 생성
   export GRADLE_HOME=/opt/gradle/gradle-7.4.2
   export PATH=${GRADLE_HOME}/bin:${PATH}
   $ sudo chmod +x /etc/profile.d/gradle.sh
   $ source /etc/profile.d/gradle.sh
   $ gradle -v

10. Jenkins 빌드 도구 설정
11. GitHub Webhook 설정



## 2. CD (Continuous Deployment)

**지속적 배포**를 뜻하며, 테스트까지 완료된 소스 코드를 개발 환경과 운영 환경에 배포하는 것을 의미한다. ArgoCD는 GitOps Repository에 저장된 메니페스트가 쿠버네티스 운영 환경에도 똑같이 반영하는 역할을 한다.

![ArgoCD](https://www.cncf.io/wp-content/uploads/2022/08/image1-31.png)

<img width="1259" alt="image" src="https://github.com/davidhyun/k8s-practice/assets/64063767/6224ac43-5eae-4b41-8c3e-82a3d123ec51">



### 2.1. Sync Method

- Push : 트리거에 따라서 배포 프로세스가 실행되는 방식

- Pull : 이미지를 배포할 때 툴(**ArgoCD**)이 자동으로 변경된 부분을 인식해서 쿠버네티스에 배포



### 2.2. ArgoCD 특징

- 지정된 환경에 어플리케이션을 자동으로 배포하고 관리
- 다수의 클러스터를 단일의 환경에서관리
- RBAC 기반의 권한 부여
- GitOps Repository를 사용하므로 백업 및 롤백 가능
- 어플리케이션 리소스의 상태 분석
- 상태(추가 및 수정 등) 동기화



### 2.3. Install ArgoCD

```bash
# k8s-master
# ArgoCD 네임스페이스 생성
$ kubectl create namespace argocd

# 클러스터에 ArgoCD 설치
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 외부 접속가능하도록 서비스 타입을 LoadBalancer로 설정
$ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# ArgoCD CLI 설치
$ VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
$ sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
$ sudo chmod +x /usr/local/bin/argocd

# Secret PW 인코딩 확인 (admin/uC-soxv8ZMmWYUaB)
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Port 확인 (http:30641/https:31113)
# 방화벽 규칙 생성 : 30641,31113 port allowed from AP IP
$ kubectl get svc -n argocd argocd-server

# ArgoCD CLI Login
$ argocd login localhost:30641

# Change Password (admin/argocd123!)
$ argocd account update-password
```

- ArgoCD Repository 설정
- GitHub 파일 생성
- ArgoCD App 배포 설정 (Sync 확인)