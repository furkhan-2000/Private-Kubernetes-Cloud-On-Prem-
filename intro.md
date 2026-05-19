## **1. Pre‑Requisites (Must be done on ALL nodes)**  
These are the base OS requirements before Kubernetes can even start.

### **1.1 OS Selection**
Choose a **stable, supported Linux version** (Ubuntu 22.04/24.04 or RHEL 8/9/10).  
Avoid outdated kernels because CNI modules may fail.

and this modern versions uses nftables, not the legacy iptables 
 you can chnage this if you want to switch back to iptbales 

### **1.2 Static IP**
Every node must have a **fixed IP** so kubelet, API server, and CNI routing remain stable.

### **1.3 Unique Hostname**
Each node must have a unique hostname so Kubernetes can register nodes correctly.

### **1.4 Disable Swap (Mandatory)**
Swap must be disabled permanently on all nodes.

**Reason:**  
Swap must be off because Kubernetes needs **real RAM**, not slow disk memory.  
If swap is on, the kernel moves pod memory to disk → pods freeze, crash, or go NotReady.  
Kubelet also cannot track real memory, so scheduling becomes wrong.  
**In short: swap confuses Kubernetes and makes nodes unstable.**

### **1.5 Network Interfaces**
Use **two NICs**:
- **NAT/Bridge** → Internet access (package downloads, image pulls)
- **Private/Internal** → Kubernetes cluster communication

---

## **2. Kernel Modules & Sysctl (Networking Requirements)**  
These settings allow Kubernetes networking to function.

### **2.1 Load Required Kernel Modules**
- **br_netfilter** → lets iptables see bridged pod traffic  
- **overlay** → required for container overlay filesystem

Kubernetes requires two kernel modules to be loaded:

- **br_netfilter** → allows pod‑to‑pod and service traffic to pass through iptables; without it, CNI networking breaks.  
- **overlay** → enables the container overlay filesystem used by containerd/runc.

These modules must load at boot, so they are placed in `/etc/modules-load.d/`.

Kubernetes also needs **IP masquerading** because pod IPs are internal and not routable on real networks. Masquerading rewrites pod traffic to the node’s IP so packets can leave the node and return correctly. Without masquerading, pod traffic to other nodes or the internet fails. Firewalld must enable masquerading on both master and worker nodes to ensure stable pod, node, and external communication.

---

If you want, I can now add the **exact commands** for loading modules and enabling masquerading.

### **2.2 Enable IP Forwarding**
Linux blocks forwarding by default.  
Kubernetes needs forwarding for:
- pod → node  
- node → node  
- pod → internet  

So enable:
- `net.ipv4.ip_forward = 1`
- bridge nf‑call rules

These must be permanent via `/etc/sysctl.d/k8s.conf`.

---

#   delete UFW ( uncomplicated firewalld and in place of that install firewalld)

 Here is the **clean, corrected, production‑grade note** for your firewall section — short, simple, and written exactly like real infra documentation.

---

# **Firewall Choice: Replace UFW with Firewalld (Clean Notes)**

UFW is Ubuntu’s simple firewall meant for desktops and small setups, but it is not suitable for Kubernetes or production environments. Firewalld is a **zone‑based, enterprise‑grade firewall** used in RHEL systems, and it provides far better control, flexibility, and security.

Firewalld supports granular rules, network zones, rich rules, protocol‑level filtering, and dynamic updates without restarting services — all of which are required for Kubernetes networking, CNI traffic, and node‑to‑node communication. Because of this, UFW should be removed and replaced with firewalld on all nodes to ensure consistent, production‑level security and proper handling of Kubernetes ports and protocols.

---

## **3. Container Runtime (CRI)**
Kubernetes needs a runtime before kubelet can start.

### **3.1 Use containerd (recommended)**
- Native CRI runtime  
- Lightweight  
- Docker is deprecated for Kubernetes

### **3.2 runc**
Low‑level engine used by containerd to run containers.

**Flow:**  
Kubernetes → containerd → runc → container

### **3.3 CNI Plugins**
Choose one:
- **Calico** (best for production)
- Flannel
- Cilium

---

## **4. Install Kubernetes Packages**
Install on **all nodes**:
- **kubeadm** → cluster bootstrap tool  
- **kubelet** → node agent  
- **kubectl** → CLI (only needed on master/Bastion)

---

## **5. Control Plane Setup (Master Node)**

### **5.1 Initialize Cluster**

This creates:
- API server  
- etcd  
- scheduler  
- controller manager  
- kubelet registration  

Node will show **NotReady** until CNI is installed.

### **5.2 Configure kubectl**
Copy admin.conf to user’s home:
- `~/.kube/config`

---

## **6. Install CNI (Networking Layer)**  
Apply Calico (recommended):


This enables:
- Pod IP assignment  
- Routing  
- Network policies  

---

## **7. Join Worker Nodes**
On each worker:
- Run the join command from master  

---

## **8. Verification**
Check cluster health:

- Nodes Ready  
- Pods Running  
- Swap disabled  
- containerd running  
- CNI OK  
- kubelet active  

---

