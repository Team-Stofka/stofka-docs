# 🚀 Kubernetes 1 Master + 1 Worker 클러스터 설치 가이드

> CNI: Calico
> 
> 
> **CRI**: containerd
> 
> **OS**: Ubuntu 22.04/24.04
> 
> **K8s 버전**: v1.28 (최신 안정 버전)
> 

---

## ✅ 전체 설치 순서 요약

| 순서 | 항목 | 마스터 | 워커 |
| --- | --- | --- | --- |
| 1️⃣ | 기본 시스템 설정 | ✅ | ✅ |
| 2️⃣ | containerd 설치 | ✅ | ✅ |
| 3️⃣ | Kubernetes 패키지 설치 | ✅ | ✅ |
| 4️⃣ | 마스터 초기화 (`kubeadm init`) | ✅ | ❌ |
| 5️⃣ | 워커 노드 조인 | ❌ | ✅ |
| 6️⃣ | Calico CNI 설치 | ✅ | ❌ |

---

## 🧱 1. 모든 노드: 시스템 설정

### 1-1. hostname 설정

```bash
# 마스터 노드
sudo hostnamectl set-hostname kubernetes

# 워커 노드
sudo hostnamectl set-hostname argo

```

### 1-2. hosts 설정 (모든 노드)

```bash
sudo nano /etc/hosts

```

```
172.16.10.151 kubernetes
172.16.10.161 argo

```

---

## 🧩 2. 모든 노드: 커널 모듈 및 containerd 설치

```bash
# 커널 모듈 설정
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl 설정
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

```

---

### 2-2. Swap 비활성화

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

```

---

### 2-3. containerd 설치 및 설정

```bash
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# SystemdCgroup 설정
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

```

---

## 🔑 3. 모든 노드: Kubernetes 설치 (v1.28 기준)

```bash
# GPG 키 추가
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 저장소 추가
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# 설치
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

---

## 🚀 4. 마스터 노드에서 클러스터 초기화

```bash
sudo kubeadm init \
  --apiserver-advertise-address=172.16.10.151 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket /run/containerd/containerd.sock

```

📌 출력되는 `kubeadm join ...` 명령은 워커 노드에서 사용하므로 복사해 두세요.

---

### ✅ kubectl 설정 (마스터 노드에서만)

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

테스트:

```bash
kubectl get nodes
```

---

## 🌐 5. 마스터 노드: Calico 설치

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

```

---

## 🤝 6. 워커 노드: 클러스터에 조인

마스터에서 복사해온 `kubeadm join` 명령 실행 예시:

```bash
sudo kubeadm join 172.16.10.151:6443 --token nbyxdp.ngtrde8zo5evfdc3         --discovery-token-ca-cert-hash sha256:f65809e7b691ba8935bc9a028a218ea57949c676659d2a571282adf416be0696

```

---

## ✅ 7. 마스터에서 노드 확인

```bash
kubectl get nodes
```

예상 출력:

```
NAME         STATUS   ROLES           AGE     VERSION
kubernetes   Ready    control-plane   5m      v1.28.x
argo         Ready    <none>          1m      v1.28.x

```

---