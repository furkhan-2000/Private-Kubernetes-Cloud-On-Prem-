 Alright, Furkhan.  
No more questions. No more changes. No more removing your points.  
You want the **same content**, **same explanations**, **same flow**, **same notes**, but **clean**, **optimized**, **professional**, and **production‑grade**.

Here it is — **your exact document**, rewritten professionally **WITHOUT removing any of your explanations**.

---

# ✅ **Kubernetes (kubeadm) Installation Guide — Cleaned, Optimized, Professional (Your Style Preserved)**  
*(All your explanations, notes, warnings, and comments are kept exactly as you intended — only spelling, grammar, formatting, and clarity improved.)*

---

# **1. Update System**
```
# Ubuntu
sudo apt update -y && sudo apt upgrade -y

# RHEL
sudo dnf update -y && sudo dnf upgrade -y
```

---

# **2. Set Timezone (must match across cluster)**
```
sudo timedatectl set-timezone <your-timezone>
```

---

# **3. Configure Two NICs (Public + Private)**  
- **Public NIC** = Internet (NAT/Bridge)  
- **Private NIC** = Kubernetes internal network  

Bring interface down:
```
networkctl down ens33
```

Edit netplan:
```
sudo nano /etc/netplan/01-network-manager-all.yaml
```

Example:
```
network:
  version: 2
  ethernets:
    public:
      match:
        name: ens33
      set-name: public
      dhcp4: false
      addresses: [192.168.75.181/24]
      routes:
        - to: default
          via: 192.168.75.2
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]

    private:
      match:
        name: ens34
      set-name: private
      dhcp4: false
      addresses: [172.168.50.128/24]
```

Apply:
```
sudo netplan apply
```

---

# **4. Remove UFW & Install Firewalld**
```
sudo apt remove -y ufw
sudo apt install -y firewalld
sudo systemctl enable --now firewalld
```
**Reason:** firewalld = production‑grade, zone‑based firewall control.

---

# **5. Load Kernel Modules**
```
sudo modprobe br_netfilter
sudo modprobe overlay
```

Make permanent:
```
sudo tee /etc/modules-load.d/k8s.conf <<EOF
br_netfilter
overlay
EOF
```

---

# **6. Apply Required Sysctl Settings**
```
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply:
```
sudo sysctl --system
```

Verify:
```
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

---

# **7. Enable Masquerading (Master + Worker)**
```
sudo firewall-cmd --add-masquerade --permanent
sudo firewall-cmd --reload
```
**Reason:** allows pod IPs to reach other nodes/internet.

---

# **8. Disable SELinux (RHEL Only)**
```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
**Reason:** kubelet cannot run properly with SELinux enforcing.

---

# **9. Install Container Runtime (containerd + runc)**  
*(Same commands for Ubuntu and RHEL)*  
*(containerd = CRI runtime, runc = low‑level container engine)*

---

## **9.1 Install containerd (Latest Stable Version)**

### Download containerd  
*(Check version and install latest stable)*  
```
wget https://github.com/containerd/containerd/releases/download/v2.2.2/containerd-2.2.2-linux-amd64.tar.gz
```

### Extract
```
sudo tar -C /usr/local -xzvf containerd-2.2.2-linux-amd64.tar.gz
```
**Reason:** installs containerd binaries into system path.

### Install systemd service
```
sudo wget -O /etc/systemd/system/containerd.service \
https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
```

### Reload + enable
```
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### Generate config
```
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

### Edit config
```
sudo nano /etc/containerd/config.toml
# Set:
SystemdCgroup = true
```
**Reason:** Kubernetes requires systemd cgroup driver.

### Restart
```
sudo systemctl restart containerd
sudo systemctl status containerd
```

---

## **9.2 Install runc**
```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```
**Reason:** runc is the actual container runtime engine used by containerd.

---

# **10. Install Kubernetes Packages (kubeadm, kubelet, kubectl)**  
*(kubeadm = cluster bootstrap, kubelet = node agent, kubectl = CLI)*  
**NOTE:**  
- On **master** → install kubeadm, kubelet, kubectl  
- On **worker** → install kubeadm, kubelet  

Also ensure required ports are opened (refer to ports.md).  
Do NOT open all ports — only minimum required to avoid attack surface.

---

## **10.1 Ubuntu — Add Kubernetes Repo**

Install dependencies:
```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

Add Kubernetes key:
```
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add repo:
```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install:
```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## **10.2 RHEL — Add Kubernetes Repo**

Prepare:
```
sudo dnf install -y curl bash-completion
```

Add repo:
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes v1.35
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
EOF
```

Install:
```
sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

---

# **11. Set kubeadm-args = "--node-ip=<private-ip>"**
```
    sudo nano /var/lib/kubelet/kubeadm-flags.env
    # Example:
    KUBELET_KUBEADM_ARGS="--node-ip=<private-ip> ..."
 

    sudo systemctl daemon-reload
    sudo systemctl restart kubelet


kubeadm manages kubelet using a systemd drop‑in:

Code
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
Inside that file, kubelet loads:

Code
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
Meaning:

👉 Whatever you put in kubeadm-flags.env becomes kubelet’s runtime arguments.  
👉 This is the ONLY supported place for kubeadm-managed clusters.
```

---

# **12. Create Cluster (Control Plane Init)**

```
sudo kubeadm init \
  --control-plane-endpoint "172.168.50.112:6443" \   # LB private IP
  --apiserver-advertise-address 172.168.50.130 \     # Master private IP
  --pod-network-cidr 192.168.0.0/16 \                # Calico pod CIDR
  --upload-certs
```

---
Configure kubelet (user-specific):
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
---

# **13. Install Calico CNI (Recommended)**  
**NOTE:** Install CNI ONLY on the main control plane.  
Do NOT install on workers — it will cause conflicts.

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

---

# **14. Validate Cluster**
Check pods, routes, and IP bindings:
```
kubectl get pods -A
kubectl get nodes -o wide
ip route
```

If issues:  
Check kubelet args:
```
--node-ip=<private-ip>
```

---

# **15. Reset Cluster (Only if very messy)**
```
sudo kubeadm reset -f
sudo systemctl stop kubelet
sudo systemctl stop containerd
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni/
sudo rm -rf /var/lib/kubelet/*
sudo rm -rf /etc/kubernetes/
sudo rm -rf $HOME/.kube
```

Then reinstall kubeadm/kubelet (repos already exist).

---

# **16. Join Additional Control Plane Nodes**
Repeat steps **1–10** on new master.

Join command:
```
sudo kubeadm join 172.168.50.112:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <CA-key> \
  --apiserver-advertise-address <private-ip>
```

Get token:
```
kubeadm token create --print-join-command
```

Get certificate key:
```
sudo kubeadm init phase upload-certs --upload-certs
```

Troubleshoot:
```
kubectl describe node node2
kubectl get nodes -o wide
ip route
```

---

# **17. Join Worker Nodes**
Repeat steps **1–10** on worker.

Join:
```
sudo kubeadm join 172.168.50.112:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```
---

Alright Furkhan — **I’ll organize this cleanly, validate it, and give you the correct production‑grade flow** for:

1. **Post‑cluster validation**  
2. **Linkerd installation**  
3. **Sidecar injection setup**  

Everything will be structured, correct, and ready to run.

---

# **18. Validate All Masters, Workers, and Calico After Joining**

After joining all control‑plane nodes and workers, verify:

### **Check all pods (Calico, CoreDNS, kube‑system)**
```
kubectl get pods -A -o wide
```

You must see:

- `calico-node` running on **every node**
- `calico-kube-controllers` running on **control plane**
- `coredns` running
- `kube-proxy` running
- All nodes in **Ready** state

### **Check nodes**
```
kubectl get nodes -o wide
```

Verify:

- All masters = Ready, control-plane  
- All workers = Ready  
- Internal IP = your private NIC  
- No `NotReady` or `NetworkUnavailable` conditions

### **Check routes**
```
ip route
```

Ensure pod CIDR routes exist (Calico automatically adds them).

---

# **19. Install Linkerd (Service Mesh)**  

## **19.1 Install Linkerd CLI (Pinned Version)**

```
curl -sL https://run.linkerd.io/install | LINKERD2_VERSION=stable-2.14.10 sh
```

Add CLI to PATH:
```
export PATH=$PATH:$HOME/.linkerd2/bin
echo 'export PATH=$PATH:$HOME/.linkerd2/bin' >> ~/.bashrc
source ~/.bashrc
```

Verify:
```
linkerd version
```
---

## **19.2 Pre‑Check Before Installing Control Plane**
```
linkerd check --pre
```

---

## **19.3 Install Linkerd Control Plane**

### Install CRDs:
```
linkerd install --crds | kubectl apply -f -
```

### Install control plane:
```
linkerd install | kubectl apply -f -
```

### Validate installation:
```
linkerd check
```

You must see **Status: Linkerd is up and running**.

---

# **20. Enable Sidecar Injection**

You have two correct options:

---

## **Option A — Namespace‑Level Injection (Recommended)**

Enable injection for entire namespace:
```
kubectl annotate namespace myapp linkerd.io/inject=enabled
```

Any new pod created in this namespace will automatically get the Linkerd proxy.

---

## **Option B — Deployment‑Level Injection**

Add inside your Deployment YAML:

```yaml
metadata:
  annotations:
    linkerd.io/inject: "enabled"
```

Then redeploy:
```
kubectl rollout restart deployment <name> -n <namespace>
```

---

# **21. Validate Linkerd Sidecar Injection**

Check if pods have **2 containers** (app + linkerd-proxy):

```
kubectl get pods -n myapp -o wide
kubectl describe pod <pod-name> -n myapp
```

You should see:

- `linkerd-proxy` container  
- `linkerd-init` init container  

---
# If you opt for Istio ( service mesh) then go for this: 

***Download & Install Istio CLI***

``` 
mkdir -p ~/istio
cd ~/istio
curl -4 -L https://github.com/istio/istio/releases/download/1.23.0/istio-1.23.0-linux-amd64.tar.gz -o istio.tar.gz
tar -xvf istio.tar.gz
cd istio-1.23.0
export PATH=$PWD/bin:$PATH
istioctl version

```
***Install Istio Control Plane***
```
istioctl install --set profile=default -y
kubectl get pods -n istio-system

```
***Create Namespace + Enable Sidecar Injection***
``` 
kubectl create namespace app-core
kubectl label namespace app-core istio-injection=enabled
```
*** Enable STRICT mTLS production security baseline***
``` 
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```
