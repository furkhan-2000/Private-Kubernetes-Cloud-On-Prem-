
## **1. Update System**
```
# Ubuntu
sudo apt update -y && sudo apt upgrade -y

# RHEL
sudo dnf update -y && sudo dnf upgrade -y
```

---

## **2. Set Timezone (must match across cluster)**
```
sudo timedatectl set-timezone <your-timezone>
```

---

## **3. Configure Two NICs (Public + Private)**  
Public = Internet (NAT/Bridge)  
Private = Kubernetes internal network  

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

## **4. Remove UFW & Install Firewalld**
```
sudo apt remove -y ufw
sudo apt install -y firewalld
sudo systemctl enable --now firewalld
```
**Reason:** firewalld = production‑grade, zone‑based control.

---

## **5. Load Kernel Modules**
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

## **6. Apply Required Sysctl Settings**
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

## **7. Enable Masquerading (Master + Worker)**
```
sudo firewall-cmd --add-masquerade --permanent
sudo firewall-cmd --reload
```

**Reason:** allows pod IPs to reach other nodes/internet.

---

if the machine is RHEL based than disable this Selinux 
```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
 kubelet cannot run properly with SELinux enforcing.
 
---

# **8. Install Container Runtime (containerd + runc)**   same commands for both rhel and ubuntu 
*(containerd = CRI runtime Kubernetes needs, runc = low‑level container engine)*

## **8.1 Ubuntu — Install containerd (Latest Version)**

### **Download containerd**   ~ check the version and then install the latest and stbale version 
```
wget https://github.com/containerd/containerd/releases/download/v2.2.2/containerd-2.2.2-linux-amd64.tar.gz
```
 get the latest stable containerd (Ubuntu repo version is outdated).

### **Extract**
```
sudo tar Cxzvf /usr/local containerd-2.2.2-linux-amd64.tar.gz
```
 installs containerd binaries into system path.

### **Install systemd service**
```
sudo wget -O /etc/systemd/system/containerd.service \
https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
```
 enables containerd to run as a system service.

### **Reload + enable**
```
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```
 start containerd and enable auto‑start on boot.

### **Generate config**
```
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```
 creates the main config file containerd uses.

### **Edit config**
```
sudo nano /etc/containerd/config.toml
# Set:
SystemdCgroup = true
```
 Kubernetes requires systemd cgroup driver.

### **Restart**
```
sudo systemctl restart containerd
sudo systemctl status containerd
```
 apply config changes and verify containerd is healthy.
---

## **8.2 Install runc (Ubuntu + RHEL)**
```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```
 runc is the actual container runtime engine used by containerd.

---

#   **9. Install Kubernetes Packages (kubeadm, kubelet, kubectl)**  
*(kubeadm = cluster bootstrap, kubelet = node agent, kubectl = CLI)*

## **9.1 Ubuntu — Add Kubernetes Repo**

### **Install dependencies**
```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
```
 required to add external repositories securely.

### **Add Kubernetes key**
```
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
 verifies Kubernetes packages are trusted.

### **Add repo**
```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```
 tells apt where to download kubeadm/kubelet/kubectl.

### **Install**
```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
sudo apt-mark hold kubelet kubeadm kubectl    ### optional 
```
 installs Kubernetes binaries and prevents accidental upgrades.

---

## **9.2 RHEL — Add Kubernetes Repo**   RHEL BASED 

### **Prepare**
```
sudo dnf install -y curl bash-completion
```
 required for repo setup and shell completion.

### **Add repo**
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
 adds official Kubernetes RPM repository.

### **Install**
```
sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```
installs Kubernetes components and starts kubelet.

---