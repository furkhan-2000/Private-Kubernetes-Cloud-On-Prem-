## **1. Wrong IP on Worker Node**
  
Worker had two NICs, and kubelet was binding to the wrong (public) IP because Linux default route pointed there. Kubernetes requires kubelet to advertise the correct internal IP, otherwise master and worker cannot communicate. We fixed it by forcing kubelet to use the private IP and reloading systemd so the change actually applied. Then we deleted the old node entry from the cluster so etcd forgets the wrong IP and the worker could rejoin cleanly.
  
- Worker had 2 NICs → kubelet picked wrong IP.  
- Added `--node-ip=<private-ip>` in:  
  - `/var/lib/kubelet/kubeadm-flags.env`  
  - `/etc/systemd/system/kubelet.service.d/10-node-ip.conf`  
- Ran `systemctl daemon-reload` (critical).  
- Deleted old node from master → `kubectl delete node <name>`.  
- Rejoined → node registered with correct IP.

---

## **2. CNI Config Missing on Worker**

### **Paragraph**  
Calico’s init container couldn’t write the CNI config because the worker wasn’t fully ready, causing a deadlock: kubelet needed CNI to start pods, but CNI needed kubelet to be ready. We broke the loop by manually copying the Calico CNI config from master to worker.

### **Notes**  
- Worker had no `/etc/cni/net.d/10-calico.conflist`.  
- Calico init container couldn’t run → node stuck NotReady.  
- Copied CNI config from master → worker.  
- Kubelet immediately started pod networking.

---

## **3. resolv.conf Symlink Issue (RHEL 10)**
 
Kubelet expects DNS config at `/run/systemd/resolve/resolv.conf`, but RHEL 10 doesn’t use systemd‑resolved, so the file doesn’t exist. Pods failed at sandbox creation because kubelet couldn’t find DNS. We created a symlink to `/etc/resolv.conf`, but `/run` is tmpfs and resets on reboot, so we added a systemd service to recreate the symlink automatically before kubelet starts.

- Missing: `/run/systemd/resolve/resolv.conf`.  
- Created symlink → `/etc/resolv.conf`.  
- `/run` wipes on reboot → symlink disappears.  
- Added systemd service to recreate symlink every boot.  
- Fixed pod sandbox failures.

---

## **4. Firewall Blocking Calico BGP**

Calico uses BGP to exchange routing information and IP‑in‑IP to tunnel pod traffic. Firewalld on RHEL 10 was blocking both, causing cross‑node pod traffic to fail. We opened BGP port 179 and protocol 4 (IPIP) permanently on both master and worker.

- Calico needs:  
  - **Port 179/tcp** (BGP)  
  - **Protocol 4** (IPIP)  
- Firewalld blocked both.  
- Added rules to internal zone on all nodes.  
- Reloaded firewalld → cross‑node pod traffic fixed.
---

## **1. What Happens After Enabling VXLAN**
  
When VXLAN is enabled, Calico switches fully to VXLAN overlay mode and stops using BGP and IPIP. This makes networking stable across mixed OS nodes because VXLAN works everywhere and avoids kernel/BIRD issues. Best practice is to disable IPIP once VXLAN is active so only one encapsulation method runs, giving cleaner routing and consistent performance.
 
- Calico stops using **BGP**.  
- Calico stops using **IPIP**.  
- Works on **all OS types** (Ubuntu, RHEL, Kali, etc.).  
- Networking becomes **stable + fast**.  
- No more **BIRD/BGP errors**.  
- Best practice → **VXLAN ON, IPIP OFF**.

### **Disable IPIP (correct command)**  
```
kubectl patch ippool default-ipv4-ippool --type='merge' -p '{"spec":{"ipipMode":"Never"}}'
```

---

## **2. Cluster Broke After VM Restart (Root Cause)**
  
The cluster failed after reboot because etcd was corrupted when the master VM was powered off without a clean shutdown. etcd is extremely sensitive to sudden power loss; when it corrupts, the API server cannot start, kubelet cannot register, kubectl loses access, and RBAC breaks. When you wiped the etcd folder, you also erased all cluster state, which is why your admin user lost permissions. Nothing was wrong with Calico, IPs, NICs, or workers — the root cause was **etcd corruption from abrupt shutdown**.
  
- Hard power‑off → **etcd corruption**.  
- API server fails → kubelet fails → kubectl fails.  
- RBAC lost because etcd data was wiped.  
- Not related to Calico, IP, NIC, or workers.  
- Root cause = **unclean shutdown of master**.

---

## 🟩 **What You Must NEVER Do in Production**

Production clusters must never be hard‑powered‑off, never run a single‑master setup, and never store etcd on unreliable disks. etcd requires clean shutdowns, stable storage, and regular snapshots. Real production clusters always use multiple etcd nodes to avoid total control‑plane failure.
  
- Never hard‑power‑off a master.  
- Never run **single‑master** in production.  
- Never store etcd on weak/unreliable disks.        # etcdctl snapshot save <filename>
- Always take **etcd snapshots**.  
- Always use **multi‑node etcd**.

---

## **3. Clean Shutdown / Startup Order (Prod Standard)**

A clean shutdown ensures etcd flushes data safely and kubelet stops gracefully. On startup, the master must come up first so etcd and the API server are ready before workers reconnect.  
- Shutdown safely:  
  ```
  sudo shutdown -h now
  ```
- Start **Bastion-LB-control-planes-workers-monitoring_servers**.  
- This is the correct and best practice sequence.  
- Prevents corruption, RBAC loss, and control‑plane failure.
---

## **4. Take ETCD Snapshot (Correct Command)**  
```
etcdctl snapshot save <filename>
```
---

## **5. Stale Pods After Reboot**

When the worker crashed, pods running on it became “ghost pods” in Unknown state. Kubernetes waits before cleaning them up, so they blocked new scheduling. After fixing DNS and kubelet, we force‑deleted the stale pods so Kubernetes could reschedule fresh ones.

- Worker reboot → pods stuck in Unknown.  
- Kubernetes doesn’t auto‑delete immediately.  
- Force‑deleted stale pods.  
- Scheduler created fresh pods → cluster healthy.

---

## **6. BGP + IPIP Explanation**
 
Calico uses BGP to distribute routing information between nodes (who has which pod CIDR). IPIP or VXLAN is then used to actually tunnel pod traffic across nodes. BGP is the routing brain; IPIP/VXLAN is the transport tunnel.

- **BGP** = routing info (who owns which pod network).  
- **IPIP/VXLAN** = tunnel that carries pod packets.  
- Your setup = **BGP + IPIP**.

---

## **7. LoadBalancer Issue on On‑Prem (MetalLB Need)**

Kubernetes cannot create real LoadBalancers on on‑prem because it depends on cloud providers to allocate external IPs. On bare‑metal, there is no provider, so EXTERNAL‑IP stays pending. MetalLB acts as the missing LoadBalancer provider and assigns IPs from your LAN, making on‑prem behave like cloud.

- On‑prem has no cloud LB provider → EXTERNAL-IP = pending.  
- Kubernetes cannot allocate external IPs by itself.  
- MetalLB provides LB functionality for bare‑metal.  
- Assigns IPs from your LAN → services get real external IPs.
