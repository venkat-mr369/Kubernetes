```bash
gcloud compute instances create ctlplane node1 node2 \
  --machine-type=e2-medium \
  --image-family=ubuntu-2004-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=10GB \
  --tags=k8s-node \
  --network=default \
  --zone=us-central1-a

gcloud compute firewall-rules create allow-k8s-internal \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:22,tcp:2379-2380,tcp:6443,tcp:10250,tcp:10255,tcp:10257,tcp:10259,tcp:30000-32767,tcp:179 \
  --source-tags=k8s-node \
  --target-tags=k8s-node


gcloud compute firewall-rules create allow-ssh \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=$(curl -s https://ipv4.icanhazip.com)/32 \
  --target-tags=k8s-node
```
### You’ll repeat this on all 3 VMs (ctlplane, node1, node2):

```bash
# 1. Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 2. sysctl config
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system

# 3. Install containerd
sudo apt update && sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd

# 4. Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# 5. Install Kubernetes components
sudo apt install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor | sudo tee /usr/share/keyrings/k8s.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/k8s.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
### Initialize Cluster on master/control_plane node
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=1.29.15
```
### To start using your cluster, you need to run the following as a regular user:
```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Alternatively, if you are the root user, you can run:
```bash
  export KUBECONFIG=/etc/kubernetes/admin.conf
```
### Install Calico CNI
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```
Join Worker Nodes (node1, node2)
### Run this on the master
```bash
kubeadm token create --print-join-command
```

output:- (from root user), Generated output execute on node1 and node2 for joining into cluster from root user
```bash
kubeadm join 10.128.0.9:6443 --token zvv9li.tp4qxxifehlzz4l9 --discovery-token-ca-cert-hash sha256:e03c28af73b531ff11a314eb3147b526b9b59028d96320867f63b82e93882e55
```
### Verify Cluster from Master
```bash
kubectl get nodes
```
```bash
gcpuser@ctlplane:~$ kubectl get nodes
NAME       STATUS   ROLES           AGE     VERSION
ctlplane   Ready    control-plane   7m25s   v1.29.15
node1      Ready    <none>          119s    v1.29.15
node2      Ready    <none>          21s     v1.29.15
```

```bash
gcloud compute instances stop ctlplane node1 node2 --zone=us-central1-a
```

```bash
gcloud compute instances start ctlplane node1 node2 --zone=us-central1-a
```

