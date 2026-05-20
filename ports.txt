# **Kubernetes Ports & iptables Notes**

## 🔹 **Ports = communication endpoints**
- **IP = house address**  
- **Port = room number**  
- Kubernetes = many rooms talking continuously.

---

# ⭐ **Why ports matter in Kubernetes**

### **Worker → API Server (6443)**
If **6443** is blocked:
- Node **does not join**
- Pods **fail to schedule**
- `kubectl` **fails**
- Networking **breaks**

---

# ⭐ **Why you switched nftables → iptables**

### **Reason**
Most Kubernetes components were originally built around **iptables**, not nftables.

### **Real production issues with nftables**
- Pod networking issues  
- DNS failures  
- NodePort problems  
- kube‑proxy routing bugs  

### **Therefore**
Many production clusters still use:

➡️ **iptables‑legacy**  
for stability and predictable behavior.

### **How to switch firewalld backend**
```
sudo vi /etc/firewalld/firewalld.conf
FirewallBackend=iptables
```

---

If you want, I can now compress this into a **single one‑page cheat sheet**.
---
**PORTS NEEDED ON EACH NODES** :

![bastion](images/1.png)
![bastion](images/4.png)
![bastion](images/2.png)
![bastion](images/3.png)
![bastion](images/5.png)
![bastion](images/6.png)

---
![ports](images/ports.png)
---
