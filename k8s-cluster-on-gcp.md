# K8S-Cluster-on-GCP

> 클라우드 상에 직접 클러스터 구성 (GCP에서 CaaS로 제공하는 GKE을 사용하지 않음)

```bash
# 인스턴스 접속
$ ssh -i ~/.ssh/myproject choihyunho317@${External-IP}

# root 계정 전환
$ sudo su -

# 우분투 기본 패키지 설치
$ sudo apt-get update && apt-get upgrade
$ sudo apt-get install -y net-tools

# 스왑메모리 비활성화 (SWAP 메모리는 쿠버네티스의 관리 영역이 아님. 활성화시 kubelet 비정상 작동 가능)
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 도커 설치
# Add Docker's official GPG key:
$ sudo apt-get install ca-certificates curl gnupg gpg lsb-release
$ sudo install -m 0755 -d /etc/apt/keyrings
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update

# Install docker runtime
$ sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
$ sudo usermod -aG docker $USER # on user shell

# cgroup 드라이버 설정 (systemd 사용요구)
$ sudo docker info
$ sudo vi /etc/docker/daemon.json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m"
    },
    "storage-driver": "overlay2"
}
$ sudo mkdir -p /etc/systemd/system/docker.service.d
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker

# 쿠버네티스 설치 키 및 레포지토리 등록 (v1.29.1)
$ sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
$ sudo apt-get update
$ sudo apt-get install -y apt-transport-https

# 쿠버네티스 설치 v1.29.1 (kubelet, kubeadm, kubectl, kubernetes-cni)
$ sudo apt-get install -y kubeadm kubelet kubectl kubernetes-cni
$ sudo apt-mark hold kubelet kubeadm kubectl # 버전 고정

# containerd 설정 변경
# https://terianp.tistory.com/177 (6443 refused error trouble-shooting)
$ containerd config default | tee /etc/containerd/config.toml
$ sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml  
$ systemctl restart containerd
$ systemctl restart kubelet

# 마스터노드 초기 설정 (마스터 노드에서만)
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=${MasterLocalIP}

# root 계정에서 그대로 수행
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ sudo systemctl restart kubelet

# 클러스터 상태 확인 (flannel-cni 설치로 coredns pending 상태 해결)
$ sudo kubectl get pods --all-namespaces
$ sudo kubectl get nodes

# 노드간 통신 네트워크 설정 (마스터노드에서 flannel 설치)
$ sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

```bash
# 워커노드에서 실행
$ kubeadm join {MasterLocalIP}:6443 --token sm35pl.5puy2ms7r3x40s8g \
	--discovery-token-ca-cert-hash sha256:c0d8eb32d6d120990fb67f15dfdc76220af810a71cb4d5382c6ceb66cb8d5d51
```