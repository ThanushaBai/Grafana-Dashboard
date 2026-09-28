# Grafana Dashboard — HashiCorp Cluster Monitoring

A standardized, reusable Grafana dashboard suite for monitoring **Nomad**, **Consul**, and **Vault** clusters — built as part of the DEV-596 task under the Grafana Dashboard Creation initiative.

The stack is fully containerized with Docker Compose and includes Prometheus for metrics collection, Loki for log aggregation, and Node Exporter for host-level metrics.

---

## 🏛️ Architecture Overview

```text
AWS EC2 (Ubuntu 22.04 / t2.micro)
│
└────────────┴────────────┐
     │ Monitoring Server │
     │  (Docker Compose) │
     └────────────┬────────────┘
                  │
     ┌────────────┬────────┴────────┬────────────────┐
     │            │                 │                │
┌────▼─────┐ ┌─────▼──────┐ ┌─────▼─────┐ ┌──────▼──────┐
│ Grafana  │ │ Prometheus │ │    Loki   │ │Node Exporter│
│  :3000   │ │   :9090    │ │   :3100   │ │    :9100    │
└──────────┘ └─────┬──────┘ └───────────┘ └──────┬──────┘
                   │ ▲                            │
                   │ │ Scrapes metrics            │
                   └──────────────────────────────┘
```

Grafana reads from Prometheus (metrics) and Loki (logs)

Prometheus scrapes Node Exporter + itself

Node Exporter exposes EC2 host metrics on :9100

### **Data flow:**

* `Node Exporter` runs as a container and exposes host metrics (CPU, memory, disk, network) on port `9100`
* `Prometheus` scrapes Node Exporter at 30-second intervals
* `Grafana` is provisioned with two datasources:

  * `grafanacloud-prom` (Prometheus) — for metrics
  * `grafanacloud-logs` (Loki) — for log querying
* Dashboards for Nomad, Consul, and Vault are imported manually into Grafana and query Prometheus using pre-validated PromQL

---

## 📁 Repository Structure

```text
Grafana-Dashboard/
├── README.md
├── docker-compose.yml
├── prometheus.yml
├── .gitignore
├── alerts/
│   └── critical-alerts.yaml # Prometheus alerting rules
├── dashboards/
│   ├── nomad-dashboard.json # DEV-598 — Nomad cluster health
│   ├── consul-dashboard.json # DEV-599 — Consul cluster health
│   └── vault-dashboard.json # DEV-600 — Vault health
├── docs/
│   ├── DEV-597_Metrics_Research.md # Prometheus metrics research
│   └── screenshots/ # Deployment + dashboard proofs
│       ├── 01-ec2-instance-running.png
│       ├── 02-stack-containers-running.png
│       ├── 03-prometheus-targets.png
│       ├── 04-grafana-datasources.png
│       ├── 05-nomad-dashboard.png
│       ├── 06-consul-dashboard.png
│       ├── 07-vault-dashboard.png
│       └── 08-alert-rules.png
└── provisioning/
    ├── datasources/
    │   └── datasources.yaml # Prometheus + Loki datasources
    └── dashboards/
        └── dashboards.yaml # Dashboard provider config
```

---

## 📋 Task Coverage

| Task ID     | Deliverable                                      | Location                           |
| ----------- | ------------------------------------------------ | ---------------------------------- |
| **DEV-597** | Prometheus metrics research (Nomad/Consul/Vault) | `docs/DEV-597_Metrics_Research.md` |
| **DEV-598** | Nomad cluster health dashboard                   | `dashboards/nomad-dashboard.json`  |
| **DEV-599** | Consul cluster dashboard                         | `dashboards/consul-dashboard.json` |
| **DEV-600** | Vault dashboard                                  | `dashboards/vault-dashboard.json`  |
| **DEV-601** | Alerting rules for critical metrics              | `alerts/critical-alerts.yaml`      |
| **DEV-596** | Parent — reusable template + documentation       | This README + repo structure       |

---

## ⚙️ Prerequisites

### **Port Matrix**

| Service           | Port | Direction | Purpose                 |
| ----------------- | ---- | --------- | ----------------------- |
| **Grafana**       | 3000 | Inbound   | Web UI access           |
| **Prometheus**    | 9090 | Inbound   | Metrics query UI + API  |
| **Loki**          | 3100 | Inbound   | Log ingestion + query   |
| **Node Exporter** | 9100 | Internal  | Scraped by Prometheus   |
| **SSH**           | 22   | Inbound   | Instance administration |

### **Security Group Rules**

Restrict all inbound traffic to **your IP** (`<your-ip>/32`) except where noted:

* `22` (SSH) — your IP
* `3000` (Grafana) — your IP
* `9090` (Prometheus) — your IP
* `3100` (Loki) — your IP (only needed if shipping logs from remote nodes)

### **Instance Requirements**

| Resource      | Minimum                     | Recommended      |
| ------------- | --------------------------- | ---------------- |
| Instance type | `t2.micro` (with 2 GB swap) | `t3.medium`      |
| RAM           | 1 GB + 2 GB swap            | 4 GB             |
| Disk          | 20 GB gp3                   | 30 GB gp3        |
| OS            | Ubuntu 22.04 LTS            | Ubuntu 22.04 LTS |

> **Note:** On `t2.micro`, the swap file is essential. Loki and Prometheus can exceed 1 GB RAM under load.

---

## 🚀 Deployment Guide

### **Phase 1 — Prepare the Instance**

**1. SSH into the EC2 instance:**

```bash
ssh -i "path/to/your-key.pem" ubuntu@<instance-public-ip>
```

**2. Add swap space (mandatory for t2.micro):**

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
```

**3. Install Docker and Git:**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-v2 git
sudo usermod -aG docker ubuntu
newgrp docker
```

**4. Verify installation:**

```bash
docker --version
docker compose version
git --version
```

---

### **Phase 2 — Deploy the Monitoring Stack**

**1. Clone the repository:**

```bash
git clone https://github.com/<your-username>/Grafana-Dashboard.git
cd Grafana-Dashboard
```

**2. Review the configuration files:**

| File                                        | Purpose                          |
| ------------------------------------------- | -------------------------------- |
| `docker-compose.yml`                        | Defines the 4-container stack    |
| `prometheus.yml`                            | Scrape targets for Prometheus    |
| `provisioning/datasources/datasources.yaml` | Grafana datasource auto-config   |
| `provisioning/dashboards/dashboards.yaml`   | Grafana dashboard auto-load path |

**3. Launch the stack:**

```bash
docker compose up -d
```

**4. Verify all containers are running:**

```bash
docker compose ps
```

Expected output — 4 containers Up:

```text
NAME            IMAGE                       STATUS
grafana         grafana/grafana:11.3.0      Up
loki            grafana/loki:latest         Up
node-exporter   prom/node-exporter:latest   Up
prometheus      prom/prometheus:latest      Up
```

---

### **Phase 3 — Verify the Stack**

**1. Prometheus targets — open in browser:**

```text
http://<instance-ip>:9090/targets
```

Both node and prometheus jobs should show UP.

**2. Grafana datasources — open in browser:**

```text
http://<instance-ip>:3000/connections/datasources
```

Login: `admin / admin`

You should see:

* `grafanacloud-prom` (default)
* `grafanacloud-logs`

**3. Import the dashboards manually:**

Grafana → Dashboards → New → Import → Upload JSON file → select a file from `dashboards/`.

When prompted for the datasource binding (`DS_PROMETHEUS`), select `grafanacloud-prom`.

Repeat for all three dashboards.

---

## 📊 Dashboards Included

### **Nomad Cluster Health (`nomad-dashboard.json`)**

| Panel                    | PromQL Query                                                                                              |
| ------------------------ | --------------------------------------------------------------------------------------------------------- |
| Nomad Node Status        | `count(up{job="nomad"}) by (instance)`                                                                    |
| Nomad CPU Utilization    | `sum(nomad_client_host_cpu_total) / (sum(nomad_client_host_cpu_total) + sum(nomad_client_host_cpu_idle))` |
| Nomad Memory Utilization | `sum(nomad_client_host_memory_used) / sum(nomad_client_host_memory_total)`                                |
| Nomad Disk Utilization   | `sum(nomad_client_host_disk_used) / sum(nomad_client_host_disk_size)`                                     |
| Nomad Job Allocations    | `nomad_nomad_job_summary_{running,pending,failed}`                                                        |
| Nomad Deployment Status  | `nomad_client_allocations_{blocked,migrating}`                                                            |

### **Consul Cluster Health (`consul-dashboard.json`)**

| Panel                  | PromQL Query                     |
| ---------------------- | -------------------------------- |
| Consul Service Health  | `consul_autopilot_healthy`       |
| Consul Raft Leadership | `consul_raft_leader_lastContact` |
| Consul Network Latency | `consul_raft_commitTime`         |

### **Vault (`vault-dashboard.json`)**

| Panel                           | PromQL Query                                                     |
| ------------------------------- | ---------------------------------------------------------------- |
| Vault Token Usage               | `vault_token_count`                                              |
| Vault Secret Engine Performance | `rate({__name__=~"vault_route_create_.*_count"}[5m]) + read ops` |
| Vault Audit Log Statistics      | `vault_audit_log_request + vault_audit_log_request_failure`      |

> **Note:** In this demo environment, Nomad/Consul/Vault panels display "No data" because there are no live HashiCorp clusters connected to the EC2. The panels are correctly configured and will populate automatically once the dashboards are imported into a production Grafana instance that scrapes a real HashiCorp environment.

---

## 🔔 Alerting Rules

Alert rules are defined in `alerts/critical-alerts.yaml` in Prometheus alerting format. To load them, reference the file in `prometheus.yml`:

```yaml
rule_files:
  - /etc/prometheus/rules/*.yaml
```

### **Rules Included**

| Alert                 | Condition                              | Severity |
| --------------------- | -------------------------------------- | -------- |
| NodeDown              | `up == 0 for 5m`                       | Critical |
| HighCPUUtilization    | `CPU > 85% for 10m`                    | Warning  |
| HighMemoryUtilization | `Memory > 90% for 10m`                 | Warning  |
| HighDiskUtilization   | `Disk > 85% for 10m`                   | Warning  |
| NomadJobFailures      | `Failed jobs > 0 for 5m`               | Critical |
| ConsulUnhealthy       | `consul_autopilot_healthy == 0 for 2m` | Critical |
| VaultSealed           | `vault_core_unsealed == 0 for 1m`      | Critical |
| VaultAuditLogFailures | `Audit failures > 0 for 5m`            | Warning  |

---

## 🔄 Adding a New Node / Cluster

To onboard a new HashiCorp cluster:

**1. Ensure the cluster exposes Prometheus metrics:**

| Tool   | Endpoint                              | Config                                                         |
| ------ | ------------------------------------- | -------------------------------------------------------------- |
| Nomad  | `/v1/metrics?format=prometheus`       | Enable `publish_node_metrics` and `publish_allocation_metrics` |
| Consul | `/v1/agent/metrics?format=prometheus` | Set `prometheus_retention_time`                                |
| Vault  | `/v1/sys/metrics?format=prometheus`   | Requires a token with read on `/sys/metrics`                   |

**2. Add a scrape target to `prometheus.yml`:**

```yaml
scrape_configs:
  - job_name: nomad
    static_configs:
      - targets: ['<nomad-server-ip>:4646']
        labels:
          cluster: 'prod-nomad'
```

**3. Reload Prometheus:**

```bash
docker compose restart prometheus
```

**4.** The dashboards will automatically display the new cluster's data — no dashboard edits required.

---

## 🛠️ Troubleshooting

### **Containers keep restarting**

Check logs:

```bash
docker compose logs --tail=50 <service-name>
```

Common cause on t2.micro: out of memory. Verify swap is active:

```bash
free -h
```

If Swap: 0B, re-run the swap setup in Phase 1.

### **Datasource shows "not found"**

Grafana provisioning only reads datasource configs on startup. After editing `provisioning/datasources/datasources.yaml`:

```bash
docker compose down grafana
docker compose up -d grafana
```

Then hard-refresh the browser (Ctrl + Shift + R).

### **Dashboards imported but panels are empty**

This is expected if:

* You're using the demo EC2 (no Nomad/Consul/Vault running)
* The datasource selected at import time was not `grafanacloud-prom`

Verify the panel datasource: Edit panel → Data source dropdown should show `grafanacloud-prom`.

---

## 📸 Implementation Proofs

| Description                                     | Screenshot                                               |
| ----------------------------------------------- | -------------------------------------------------------- |
| EC2 instance running                            | https://docs/screenshots/01-ec2-instance-running.png     |
| All containers up                               | https://docs/screenshots/02-stack-containers-running.png |
| Prometheus targets UP                           | https://docs/screenshots/03-prometheus-targets.png       |
| Grafana datasources connected                   | https://docs/screenshots/04-grafana-datasources.png      |
| Nomad dashboard (with live Node Exporter panel) | https://docs/screenshots/05-nomad-dashboard.png          |
| Consul dashboard                                | https://docs/screenshots/06-consul-dashboard.png         |
| Vault dashboard                                 | https://docs/screenshots/07-vault-dashboard.png          |
| Alert rules configured                          | https://docs/screenshots/08-alert-rules.png              |

---

## 📚 References

### **Nomad Metrics Reference**

### **Consul Telemetry Documentation**

### **Vault Telemetry Metrics**

### **Grafana Provisioning Docs**

### **Prometheus Alerting Rules**

---

## 👤 Author

**Thanusha Bai V — DevOps Intern**

**Task:** DEV-596 — Grafana Dashboard Creation (Parent)

---

## 📄 License

Internal use only — not for public distribution.
