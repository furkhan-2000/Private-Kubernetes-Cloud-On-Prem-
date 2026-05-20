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

# delete UFW ( uncomplicated firewalld and in place of that install firewalld)

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

#  **(NEW) Kubelet — What, Where, Why**

### **Kubelet Overview**  
Kubelet is the primary node agent responsible for running and managing pods on every Kubernetes node. It communicates with the API server, starts and stops pods, monitors container health, restarts failed containers, and sets up pod networking by invoking the CNI plugin. It also reports the node’s status back to the control plane. Without kubelet, a node cannot join the cluster, cannot run pods, and will always remain in a NotReady state.
  
- Starts/stops pods based on API server instructions.  
- Monitors containers and restarts them on failure.  
- Reports node status (**Ready/NotReady**) to the control plane.  
- Calls the **CNI plugin** to configure pod networking.  
- Keeps the node registered and healthy in the cluster.  
- Runs on **every node** as a systemd service.  
- Required for kubeadm init/join to work.  
- Without kubelet → node cannot join, pods cannot run, node stays **NotReady**.

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
---

# ✅ **TLS Bootstrap**

TLS Bootstrap is the secure process where a new kubelet proves its identity to the API server and becomes a trusted node. The kubelet uses a temporary bootstrap token to request a certificate, the control‑plane signs it, and kubelet switches to this real certificate for all future communication. This ensures only authorized nodes join the cluster and all kubelet ↔ API server traffic is encrypted.
 
- kubelet sends CSR → API server signs → kubelet becomes trusted.  
- Node registers only after certificate is issued.  
- Prevents fake/rogue nodes from joining.  
- All kubelet communication is TLS‑encrypted.  
- Required for secure production clusters.

---

#  **What survives reboot**  
- Node, kubelet, containerd, pods, cluster state → **survive reboot**.  
- Certificates → **expire**, not lost on reboot but expire over time.

---

#  **Check & Renew Certificates**  
```
kubectl certs check-expiration
kubectl certs renew <component>
```

---

# **TLS Termination**

TLS Termination means the Load Balancer or Ingress decrypts HTTPS traffic and sends plain HTTP to backend pods. This centralizes certificate management, reduces CPU load on pods, and simplifies rotation. It is the most common production setup because certificates live only on the LB/Ingress, not inside every pod.
 
- LB/Ingress decrypts HTTPS → pod gets HTTP.  
- Certificates stored only on LB/Ingress.  
- Simplifies cert rotation and management.  
- Reduces CPU load on pods.  
- Used in 90% of real production clusters.

---

# **Traffic Flow (main)**  
```
Client → HTTPS → Load Balancer → HTTP → Pod

---

# ⭐ **ETCD + RAFT + Certificates — Final Concise Notes**

## 🔹 **etcdctl (etcd‑client) — what it does**
- check etcd **health**
- check **quorum**
- check **leader**
- check **members**
- take **backups**
- view **cluster status**

---

# ⭐ **ETCD RAFT Concepts**

| Term | Meaning (simple) | Indicates | Good/Bad |
|------|------------------|-----------|----------|
| **IS LEARNER** | new node syncing | not voting yet | false = GOOD |
| **RAFT TERM** | election round | leader changes | higher = normal |
| **RAFT INDEX** | total writes stored | cluster updates | should increase |
| **APPLIED INDEX** | writes applied locally | node sync status | must match index |
| **Mismatch** | node behind | slow/unhealthy | small gap OK, big gap BAD |

---

# ⭐ **Ultra‑Short Memory Trick**
- **Learner** → new guy learning  
- **Term** → election round  
- **Index** → total writes  
- **Applied** → writes applied locally  
- **Match = healthy**, mismatch = lag  

---

# ⭐ **Commands**

### **Single master**
```
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

### **All 3 masters**
```
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://172.168.50.130:2379,https://172.168.50.139:2379,https://172.168.50.137:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

---
#  **Certificate Rule**

### **All Kubernetes certificates originate ONLY from master1.**

- master1 = **root CA + API certs + front‑proxy certs + SA keys**
- master2/master3 **reuse** these certs  
- they **do not generate new ones**
- workers **never receive certs manually**  
  - kubelet auto‑requests a signed cert from the CA on master1

### ✔ Simple rule:
**Everything is signed by master1.  
All other nodes must trust master1’s CA.**