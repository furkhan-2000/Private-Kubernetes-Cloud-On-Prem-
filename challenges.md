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
