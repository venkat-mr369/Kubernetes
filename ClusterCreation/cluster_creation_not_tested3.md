# **Step-by-Step Kubernetes Installation on Oracle Linux 9.5**  
### **Servers Configuration:**  
- **Master Node:** `192.168.17.101 (server1)`  
- **Worker Node 1:** `192.168.17.102 (server2)`  
- **Worker Node 2:** `192.168.17.103 (server3)`  

---

## **1. Prerequisites**
### **Step 1: Update System Packages on All Nodes**
```bash
sudo dnf update -y
sudo dnf install -y vim git curl wget yum-utils device-mapper-persistent-data lvm2
```

---

### **Step 2: Disable Swap (Kubernetes requires swap to be disabled)**
```bash
sudo swapoff -a
sed -i '/swap/d' /etc/fstab
```

---

### **Step 3: Set Hostnames**
Run the following on each node:

#### On **Master Node (server1)**:
```bash
sudo hostnamectl set-hostname master-node
```

#### On **Worker Node 1 (server2)**:
```bash
sudo hostnamectl set-hostname worker-node-1
```

#### On **Worker Node 2 (server3)**:
```bash
sudo hostnamectl set-hostname worker-node-2
```

Update `/etc/hosts` on **all nodes**:
```bash
echo "192.168.17.101 master-node
192.168.17.102 worker-node-1
192.168.17.103 worker-node-2" | sudo tee -a /etc/hosts
```

---

### **Step 4: Configure Firewall on All Nodes**
```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --reload
```

---

### **Step 5: Enable IP Forwarding on All Nodes**
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

---

## **2. Install Container Runtime (Containerd)**
```bash
sudo dnf install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## **3. Install Kubernetes (Kubeadm, Kubelet, and Kubectl)**
### **Step 1: Add Kubernetes Repository**
```bash
sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

### **Step 2: Install Kubeadm, Kubelet, and Kubectl**
```bash
sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

---

## **4. Initialize Kubernetes Cluster (On Master Node Only)**
```bash
sudo kubeadm init --apiserver-advertise-address=192.168.17.101 --pod-network-cidr=192.168.1.0/16
```

After successful execution, you will get a **join command**. Copy this command, as it will be needed for worker nodes.

---

## **5. Configure kubectl (On Master Node Only)**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify Kubernetes cluster:
```bash
kubectl get nodes
```

---

## **6. Install CNI (Calico) Network Plugin**
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

## **7. Join Worker Nodes to the Cluster**
Run the **join command** (copied from master setup) on **Worker Nodes**:

```bash
sudo kubeadm join 192.168.17.101:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Verify all nodes:
```bash
kubectl get nodes
```

---

## **8. Deploy a Test Application**
```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get svc
```

To access it:
```bash
curl http://<worker-node-ip>:<NodePort>
```

---

# **Conclusion**
✅ Installed Kubernetes on **Oracle Linux 9.5**  
✅ Configured **Master & Worker Nodes**  
✅ Set up **Networking & CNI**  
✅ Tested Kubernetes with **Nginx Deployment**  

