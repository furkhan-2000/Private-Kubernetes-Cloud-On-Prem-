# **Multi‑Master + LB + Quorum + kubectl Behavior**

---

## **1. Joining More Masters & Workers**
  
In an on‑prem HA cluster, adding more masters and workers simply means joining them to the same control plane and letting etcd maintain quorum. Masters sync through etcd, workers sync through the API server, and the Load Balancer provides a single stable API endpoint so all nodes stay consistent.
  
- Masters sync via **etcd**.  
- Workers sync via **API server**.  
- LB provides **one stable API IP**.  
- More masters = higher availability.  
- More workers = more compute capacity.

---

## **2. Quorum**

Quorum means **majority agreement** among masters before etcd accepts any write. This prevents split‑brain and protects cluster state. Always use an **odd number** of masters so majority is mathematically possible.

- Quorum = **>50% masters must agree**.  
- Prevents split‑brain.  
- Always **odd number** of masters (1, 3, 5).  
- etcd refuses writes without quorum.
     **Because without a majority, etcd cannot be sure which master is correct, so it blocks all writes to avoid split‑brain**
       Split‑brain = **when two masters think they are the leader at the same time and make conflicting decisions**

---

## **3. Workers & kubectl Cannot Talk to Masters Directly**
 
In HA mode, workers and kubectl never talk to masters directly. They always talk to the **Load Balancer VIP**, which forwards traffic only to healthy masters. This ensures API availability even if one master dies.

- Worker → LB → Master  
- kubectl → LB → Master  
- No direct worker → master API traffic  
- LB is the **only entry point** for API calls

---

#  **4. LB + Multi‑Master Architecture**

## **How LB Chooses a Master**
- LB health‑checks port **6443** on each master.  
- If a master is down → LB removes it.  
- Uses **Round Robin** or **Least Connections**.

## **How Worker Points to LB**
- Worker stores API endpoint in:  
  `/etc/kubernetes/kubelet.conf`  
- In HA → this IP = **LB VIP**, not a master IP.  
- Worker doesn’t care how many masters exist.

## **How LB Sends to Workers**
- LB **never** sends traffic to workers.  
- Only API traffic goes through LB.  
- Pod traffic = worker ↔ worker directly.  
- Master ↔ worker = direct (10250).

## **Simple Mental Model**
**LB = receptionist**  
**Masters = agents**  
**Workers = field staff**  
You call one number (LB), receptionist forwards to any available agent (master).

---
**NOTE:**
we should  never SSH into masters or run kubectl on them. All debugging is done through centralized logs (ELK, Loki, Splunk, Datadog) and monitoring dashboards (Prometheus, Grafana, Datadog). Masters are only accessed with restricted break‑glass SSH during emergencies. Normal engineers use dashboards, not SSH, to understand master issues.

---

#  **5. Reboot Survival Order**

After a full shutdown, the cluster only comes back cleanly if the LB starts first, then masters, then workers. This ensures the API endpoint is reachable, etcd forms quorum, and workers reconnect without errors.

1. Start **Load Balancer** first.  
2. Wait 2–3 seconds.  
3. Start **all masters** → wait for etcd + API.  
4. Start **workers** last.  
5. Cluster restores cleanly.

---

# **6. kubectl Behavior**

kubectl is only a client. It never runs on masters or workers. It sends HTTPS API requests to the LB VIP, the LB forwards to a healthy master, and the master processes the request. Masters do not need kubectl installed because they only receive API calls, not execute kubectl locally.

- kubectl = client, like a browser.  
- Sends API calls → LB VIP.  
- LB → forwards to master.  
- Master processes request via API server.  
- kubectl **never runs on masters**.

### **Flow Example**
```
kubectl → LB → Master API → etcd → Scheduler → Worker → Pod
```
