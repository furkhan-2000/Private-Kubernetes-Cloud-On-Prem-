# **kubectl on Bastion**
 
Setting up kubectl on a bastion is the standard production practice because the bastion becomes the single, controlled entry point for managing the cluster. You install kubectl on the bastion, copy a user‑specific kubeconfig from the master, and operate the entire cluster from there. In production, **kubeconfig is always per‑user, never global,** so RBAC, auditing, and access control remain clean and secure.

---

### **1. Install kubectl on Bastion**
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### **2. Copy kubeconfig from Master → Bastion**

**On Master:**
```
sudo cat /etc/kubernetes/admin.conf
```

**On Bastion:**
```
mkdir -p ~/.kube
vim ~/.kube/config   # paste content
chmod 600 ~/.kube/config
sudo chown romeo:romeo ~/.kube/config
```

### **Test**
```
kubectl get nodes
```

---

#  **Why This Is Good Practice in Production**

Using a bastion centralizes access, enforces RBAC, avoids exposing masters, and ensures all kubectl activity is logged and audited. Each engineer gets their own kubeconfig so permissions are isolated, traceable, and secure. This prevents credential sharing, accidental cluster‑wide access, and unauthorized changes.

- **Masters stay locked down** — no kubectl installed.  
- **Bastion = single controlled entry point**.  
- **Per‑user kubeconfig** = clean RBAC + audit logs.  
- **No shared credentials**.  
- **No direct SSH to masters**.  
- **All kubectl traffic goes through LB VIP**.  
- **Security teams prefer this model**.  
- **Industry standard for on‑prem + cloud clusters**.

---

kubectl belongs on the bastion, not on masters — per‑user kubeconfig + RBAC + auditing = real production security, and in real environments we never log into masters or workers; all control happens from the bastion, with SSH used only in rare critical emergencies.