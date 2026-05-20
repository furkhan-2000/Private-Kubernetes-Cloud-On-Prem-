# **kubectl → LB → Masters**

kubectl never talks to masters directly; it always sends API requests to the Load Balancer VIP. The LB checks which master is healthy and forwards the request. This is how real production clusters work: kubectl from laptop or bastion → LB VIP → healthy master. This design gives high availability, load balancing, and zero downtime even if a master dies or is upgrading.

---

#  **Why This Is Good Practice (Very Short)**  
- If **master1 dies**, LB sends traffic to master2/master3.  
- If **master2 is overloaded**, LB balances to others.  
- If **master3 is upgrading**, LB avoids it.  
- Ensures **continuous API availability**.  
- kubectl always uses **one stable endpoint** (LB VIP).  
- No engineer ever talks to masters directly.

---

#  **Production Kubernetes Flow**  
**kubectl → LB VIP → Masters → etcd → Scheduler → Workers → Pods**  
Workers also report back to masters **through the same LB VIP**.
 
- LB forwards API traffic to any healthy master.  
- Masters read/write cluster state via etcd quorum.  
- Scheduler assigns pods to workers.  
- Workers run pods (kubelet + containerd).  
- Calico handles pod routing (BGP).  
- If a master fails → LB automatically shifts traffic.


# 🟦 **Load Balancer Setup (HAProxy)**

### **LB VM Requirements**  
- RAM: **1.5 GB**  
- CPU: **1**  
- Disk: **8 GB**  
- OS: Ubuntu recommended  
- Purpose: **Only forward API traffic (6443)**

### **Install HAProxy**
```
sudo apt update
sudo apt install haproxy -y
```

### **Configure HAProxy**
```
sudo vim /etc/haproxy/haproxy.cfg
```

Add:
```
frontend kubernetes
    bind *:6443
    mode tcp
    default_backend k8s-masters

backend k8s-masters
    mode tcp
    balance roundrobin
    server master1 172.168.50.130:6443 check
    server master2 172.168.50.131:6443 check
    server master3 172.168.50.132:6443 check
```
 
- `frontend` listens on **6443** (Kubernetes API).  
- `backend` lists all masters.  
- `balance roundrobin` spreads load.  
- `check` health‑checks each master.  
- If a master fails → LB removes it automatically.

### **Restart**
```
sudo systemctl restart haproxy
```

---

# 🟧 **Bastion Host**

- Single secure entry point for all engineers.  
- All kubectl commands run **from bastion**, never from masters.  
- SSH to masters/workers only in **critical emergencies**.

### **Install Tools on Bastion**
```
vim curl wget git net-tools htop cloud-guest-utils lvm2 kubeadm kubectl
```

### **User Management (Important)**  
**Bastion needs ONLY ONE admin user.**  
Example: `romeo`

Even if nodes have different usernames:

- Master1 → amreen  
- Master2 → john  
- Worker1 → rahul  
- Worker2 → mike  

You **do NOT** recreate these users on bastion.

### **Correct Method**  
Use **one SSH key** → copy to all nodes’ `authorized_keys`.

### 🟥 **Do NOT:**  
- Do NOT create multiple users on bastion.  
- Do NOT match usernames with nodes.  

### 🟩 **Correct:**  
- One admin user on bastion.  
- That user SSHes into all nodes using keys.

---

# 🔐 **Restrict SSH Access on Masters/Workers**

Edit:
```
sudo vim /etc/ssh/sshd_config
```

Add at bottom:
```
AllowUsers romeo@192.168.75.200
```

**Meaning:**  
Only the bastion’s private IP can SSH into masters/workers.

---

#  **What This HAProxy Config Does**  
This HAProxy config turns your VM into a **TCP load balancer for the Kubernetes API**. It listens on **port 6443** and forwards all API traffic to your master nodes using **round‑robin** with **health checks**. If a master fails, HAProxy automatically removes it and sends traffic only to healthy masters.  
This is the **standard production HA pattern** used in AWS, GCP, Azure, and on‑prem clusters.

---

#  **Config Breakdown**

## **1) frontend kubernetes**
```
frontend kubernetes
    bind *:6443
    mode tcp
    default_backend k8s-masters
```

### **Meaning (Real Production)**  
- `bind *:6443` → LB listens on Kubernetes API port  
- `mode tcp` → raw TCP passthrough (API uses HTTPS)  
- `default_backend` → forwards all API traffic to master nodes  

**This is the entry point for kubectl, kubelet, and all API calls.**

---

## **2) backend k8s-masters**
```
backend k8s-masters
    mode tcp
    balance roundrobin
    server m1 172.168.50.130:6443 check
    server m2 172.168.50.139:6443 check
    server m3 172.168.50.137:6443 check
```

### **Meaning**  
- `mode tcp` → same as frontend  
- `balance roundrobin` → each request goes to the next master  
- `check` → health‑checks each master  

**If master1 dies → LB automatically sends traffic to master2 & master3.**

---

# 🟦 **Why Round Robin?**  
- Kubernetes API is **stateless**  
- Any master can handle any request  
- No sticky sessions needed  
- Even load distribution  
- Fast, simple, reliable  

This is the **industry standard** for Kubernetes control‑plane load balancing.

---

### **1) mode tcp**  
Used for:  
- Kubernetes API (6443)  
- etcd (if load‑balanced)  
- Any encrypted traffic  

**Required for Kubernetes.**

### **2) mode http**  
Used for:  
- Web apps  
- Ingress controllers  

**NOT used for Kubernetes API.**

### **3) balance roundrobin**  
Best for API servers.

Other modes (not needed for K8s):  
- `leastconn` → long‑lived connections  
- `source` → sticky sessions  
- `random` → weighted random  

---

# 🟥 **What You Do NOT Need (Important)**  
You do **NOT** need to load balance:  
- 10250 (kubelet)  
- 179 (BGP)  
- 2379/2380 (etcd)  
- HTTP mode  
- SSL termination  
- Sticky sessions  
- Any advanced routing  

**LB = 6443 only.**  
Everything else is internal between masters and workers.

---

# 🟩 **Global + Defaults Section**  
The `global` block controls HAProxy’s internal behavior:  
- Logging to `/dev/log`  
- Running as a daemon  
- Running as restricted user `haproxy`  

The `defaults` block sets:  
- `mode tcp` → required for API  
- `option tcplog` → clean TCP logs  
- Timeouts → prevent hanging connections  

**None of these affect Kubernetes traffic — they only control HAProxy’s behavior.**

---

# 🟥 **If Everything Breaks (Reset All Kubernetes Nodes)**  
Use this when cluster was initialized with wrong master IP or workers pointed to wrong API endpoint.

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

---

# 🟩 **Re‑Initialize Master1 (Correct, Validated)**  
```
sudo kubeadm init \
  --control-plane-endpoint "172.168.50.112:6443" \
  --apiserver-advertise-address 172.168.50.130 \
  --pod-network-cidr 192.168.0.0/16 \
  --upload-certs
```

Then install Calico → then join other masters.

---

# 🟦 **Master Join**

### **Step 1 — Worker join command**
```
kubeadm token create --print-join-command
```

### **Step 2 — Certificate key (for master join only)**
```
sudo kubeadm init phase upload-certs --upload-certs
```

### **Step 3 — Combine** # for joining new master to the Quorum K8s cluster
```
sudo kubeadm join 172.168.50.112:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key> \
  --apiserver-advertise-address <private-ip-of-new-master>
```

---

# 🟩 **Worker Join**  
```
sudo kubeadm join 172.168.50.112:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

# 🟦 **Check Ghost etcd Members**
```
sudo etcdctl \
  --endpoints=https://<private-ip>:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

---

# 🟩 **Important Note About init**  
`--control-plane-endpoint` **must always be the LB private IP**.  
Then join all masters using the join command.

---

# 🟥 **What Went Wrong**  
Master2 join failed at **check‑etcd** because firewall blocked:  
- **2379** (etcd client)   how masters read/write data to etcd,
- **2380** (etcd peer)     how etcd nodes sync state with each other to maintain quorum.

Join died halfway → certificates created but kubelet config missing → kubelet crash‑looped.

Master1 never had this issue because `kubeadm init` runs etcd locally → no network traffic → firewall irrelevant.

---

# ⭐ **One‑Line Summary**  
**HAProxy on port 6443 load‑balances all API traffic to healthy masters, kubectl always talks to the LB VIP, and this architecture is the real production standard for multi‑master Kubernetes clusters.**
