# **LOGGING + MONITORING STACK — PRODUCTION NOTES**

---

# 🔹 **ELK (Elasticsearch + Logstash + Kibana)**  
- Elasticsearch → stores logs  
- Logstash → processes logs  
- Kibana → dashboards/search  
- Heavy, resource‑intensive  
- Good for centralized logging + search + alerting  

---

# 🔹 **Splunk (Enterprise SIEM)**  
- Forwarder → collects logs from servers  
- Indexer → stores + indexes logs  
- Search Head → dashboards, alerts, correlation  
- Used for SIEM + threat detection  
- Paid, licensed per GB/day  

---

# **Loki + Fluent Bit + Prometheus + Node Exporter + Grafana**

### **What each component does**
- **Fluent Bit** → log collector on every node  agent
- **Loki** → central log storage  
- **Node Exporter** → system metrics on every node  
- **Prometheus** → scrapes + stores metrics  
- **Grafana** → dashboards for logs + metrics  

### **Where they run**
- **All nodes (masters, workers, bastion):**  
  - Fluent Bit  
  - Node Exporter  
- **Monitoring server:**  
  - Loki  
  - Prometheus  
  - Grafana  

### **Pipeline summary**
- Fluent Bit → sends logs → Loki  
- Node Exporter → exposes metrics → Prometheus scrapes  
- Grafana → visualizes both  

---

# ⭐ **INSTALLATION NOTES**

---

# 🔹 **Fluent Bit (ALL SERVERS)**

### Ubuntu
- Add Fluent Bit repo  
- Install package  
- Verify binary  
- Enable + start service  

### RHEL
- Add repo file  
- Install via dnf  
- Enable + start service  

### Deployment method
- Install on one server  
- Push script via rsync → bastion → all nodes  

---

# 🔹 **Node Exporter (ALL SERVERS)**

### ⭐ Why create `node_exporter` user
- Least privilege  
- No login  
- Isolated service  
- Passes security audits  
- Standard enterprise practice  

### ⭐ Why move binary to `/usr/local/bin`
- Permanent location  
- Safe for systemd  
- Not overwritten by OS updates  
- Correct place for custom executables  

### Install steps
- Create user  
- Download binary  
- Move to `/usr/local/bin`  
- Set permissions  
- Create systemd service  
- Enable + start  
- Open port **9100/tcp** on private NIC  

---

# 🔹 **Prometheus (Monitoring Server)**

### Install steps
- Create user  
- Create directories  
- Download + extract binary  
- Move binaries + config  
- Create systemd service  
- Enable + start  
- Verify: `curl localhost:9090/-/healthy`

### Add all nodes to `prometheus.yml`
- Prometheus scrapes **9100** on every server  
- Restart Prometheus after editing  

---

# 🔹 **Loki (Monitoring Server)**

### Install steps
- Create `loki` user  
- Create directories  
- Download binary  
- Move to `/usr/local/bin`  
- Create config file  
- Create systemd service  
- Enable + start  

### Fluent Bit → Loki output (ALL SERVERS)
```
[OUTPUT]
    Name   loki
    Host   <monitoring-ip>
    Port   3100
    Labels job=fluentbit,host=${HOSTNAME}
```

---

# 🔹 **Grafana (Monitoring Server)**

### Install steps
- Download `.deb`  
- Install  
- Enable + start  
- Access: `http://<ip>:3000`  
- Add Loki datasource: `http://localhost:3100`  

---

# ⭐ **PORTS**

| Service | Port | Direction |
|---------|-------|-----------|
| Prometheus | **9090/tcp** | inbound (public NIC) |
| Grafana | **3000/tcp** | inbound (public NIC) |
| Loki | **3100/tcp** | inbound (private NIC only) |
| Node Exporter | **9100/tcp** | inbound (private NIC on ALL servers) |

### Production logic
- **Loki 3100** → only monitoring server private NIC  
- **Node Exporter 9100** → all nodes private NIC  
- **Prometheus/Grafana** → only public NIC for admin access  

---

# ⭐ **Grafana Dashboards**  
- Go to **Grafana Dashboards (Grafana Labs)**  
- Copy dashboard ID  
- Import in Grafana  
- Select Prometheus/Loki datasource  
- Dashboard loads instantly with your data 
---
**Fluent-bit ( agent for logs ) installation steps**

# installing prerequisities
echo "===== INSTALLING PREREQUISITES ====="

sudo apt update
sudo apt install -y curl gnupg ca-certificates

# adding repo of FLUENT

echo "===== ADDING OFFICIAL FLUENT BIT REPO ====="

sudo mkdir -p /usr/share/keyrings

curl -fsSL https://packages.fluentbit.io/fluentbit.key \
| sudo gpg --dearmor -o /usr/share/keyrings/fluentbit.gpg

echo "deb [signed-by=/usr/share/keyrings/fluentbit.gpg] https://packages.fluentbit.io/ubuntu/noble noble main" \
| sudo tee /etc/apt/sources.list.d/fluentbit.list

#Installing fluent-bit

echo "===== INSTALLING FLUENT BIT ====="

#   installing fluent-bit 
sudo apt update
sudo apt install -y fluent-bit

#   verify installation
echo "===== VERIFY INSTALLATION ====="

which fluent-bit
fluent-bit --version

#   forwarding this setup to all server remotely through rsync 

To deploy Fluent Bit across all servers in a production‑safe way, you first install and prepare the `fluent.sh` script on **one machine**, verify it works, and then distribute it to the rest of the environment. Because internal nodes are only reachable through the bastion, the script must be copied to the bastion first. From the bastion and from the authorized user, you can securely push the same script to every master, worker, and any other node. This ensures consistent installation across the entire cluster and respects network‑access restrictions.

### **Command used to send script to bastion**
```
rsync -avz -e ssh fluent.sh bastion-user@<bastion-ip>:/home/bastion-user/
```

### **Command used on bastion to send script to any internal node**
```
rsync -avz -e ssh fluent.sh <target-user>@<target-ip>:/home/<target-user>/
```

---

FOR RHEL:: add this repo to rhel so it installed /etc/yum.repo.d/ 
# 1. Add repo
sudo tee /etc/yum.repos.d/fluent-bit.repo > /dev/null <<EOF
[fluent-bit]
name=Fluent Bit
baseurl=https://packages.fluentbit.io/centos/9
gpgcheck=1
gpgkey=https://packages.fluentbit.io/fluentbit.key
enabled=1
EOF
# 2. Refresh metadata
sudo dnf clean all
sudo dnf makecache

# 3. Install
sudo dnf install -y fluent-bit

Sudo systemctl start fluent-bit 
Sudo systemctl enable fluent-bit --now 
---
#   **Node-Exporter ( agent for logs ) installation steps**
1. **Create system user (no login)**
sudo useradd --no-create-home --shell /sbin/nologin node_exporter  

2. **Download latest stable binary**
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.11.1/node_exporter-1.11.1.linux-amd64.tar.gz

3. **Extract package**
tar -xvf node_exporter-1.11.1.linux-amd64.tar.gz

4. **Move binary to system path**
sudo mv node_exporter-1.11.1.linux-amd64/node_exporter /usr/local/bin/

5. **Set permissions**
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

6. **Create systemd service**
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target
EOF

7. **Reload systemd & start, enable**
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter

8. **Test metrics endpoint**
curl http://localhost:9100/metrics
🔥 RHEL / Firewalld:
sudo firewall-cmd --add-port=9100/tcp --permanent
sudo firewall-cmd --reload
---
# **Prometheus installation steps:**

sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus

cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.4.1/prometheus-3.4.1.linux-amd64.tar.gz

tar -xvf prometheus-3.4.1.linux-amd64.tar.gz
cd prometheus-3.4.1.linux-amd64

sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles console_libraries prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

sudo tee /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus
After=network.target
[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=15d
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus

# Verify
curl http://localhost:9090/-/healthy

**Prometheus Setup Notes:**
Prometheus only scrapes itself by default. To monitor all your servers, you must add their IPs under scrape_configs. This tells Prometheus where Node Exporter is running. Node Exporter always runs on port 9100, so Prometheus connects to that port on every server to collect CPU, RAM, Disk, Network, and system metrics.
You simply create a new job called nodes under scrape_configs and list all your server IPs. This is the standard production method for static Linux servers. After adding the IPs, restart Prometheus so it loads the new configuration.

  - job_name: "nodes"
    static_configs:
      - targets:
        - "master1-ip:9100"
        - "master2-ip:9100"
        - "master3-ip:9100"
        - "worker1-ip:9100"
        - "loadbalancer-ip:9100"
This makes Prometheus scrape all your servers
📝 Restart Prometheus
sudo systemctl restart prometheus
Prometheus will immediately start collecting metrics from all servers.
---

# **LOKI SETUP:**

**Loki 3.7.1**

sudo useradd --no-create-home --shell /sbin/nologin loki

sudo mkdir -p /etc/loki /var/lib/loki/chunks /var/lib/loki/rules

cd /tmp
curl -L -o loki.zip https://github.com/grafana/loki/releases/download/v3.7.1/loki-linux-amd64.zip
unzip loki.zip

sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki

**SYSTEMD SERVICE:**
sudo tee /etc/systemd/system/loki.service << 'EOF'
[Unit]
Description=Loki Log Aggregation System
After=network.target
[Service]
User=loki
Group=loki
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yml
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now loki
---

After this loki installed we don’t configure loki file like how we did in prometheus/prometheus.yaml on monitor server, but we need to configure fluent-bit (agent which is running on all servers). 

On all servers where this fluent bit is installed there /etc/fluentbit/fluentbit.config Fluent‑bit already collects logs from /var/log/*, so you don’t touch the INPUT or PARSER sections. You simply add an OUTPUT section pointing to your monitoring server’s private IP on port 3100, and use host=${HOSTNAME} so each server automatically tags its own hostname
here at last in output edit this file 
[OUTPUT]
    Name        loki
    Host        172.168.50.145
    Port        3100
    Labels      job=fluentbit,host=${HOSTNAME}
---

**Grafana 13.0.1**
cd /tmp
wget https://dl.grafana.com/oss/release/grafana_13.0.1_amd64.deb

sudo apt install -y ./grafana_13.0.1_amd64.deb
sudo systemctl enable --now grafana-server

Access: http://<ip>:3000 — login admin/admin
Add Loki data source: http://localhost:3100


Service	Port	Direction
Prometheus	9090/tcp	inbound
Loki	3100/tcp	inbound
Grafana	3000/tcp	Inbound
Node_exporter	9100/tcp	

On a production monitoring server, only the monitoring node must expose Loki’s ingestion port 3100, and it must be open only on the private NIC, because all other servers send logs to Loki and do not receive any. In contrast, Node Exporter’s port 9100 must be open on every server private NIC , including the monitoring server private NIC , because Prometheus scrapes metrics from all nodes, and each node must allow inbound access on 9100 through the private NIC. UI ports like 3000 (Grafana) and 9090 (Prometheus) remain open only on the public NIC for admin access. 

When you want predefined dashboards in Grafana, you get them from the official Grafana dashboards website ( Grafana dashboards | Grafana Labs ). This site provides ready‑made dashboards like Node Exporter Full, Prometheus Stats, Loki Logs, and many more. You simply copy the dashboard ID from the website and import it inside Grafana. After importing, Grafana automatically uses your Prometheus or Loki datasource, so the same predefined dashboard layout appears but filled with your own metrics and logs. This is exactly how real production teams work—no manual graph creation, just import → select datasource → done.  

