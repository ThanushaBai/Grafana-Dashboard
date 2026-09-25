# **DEV-597: Prometheus Metrics Research — Nomad, Consul, Vault**

**Task ID:** DEV-597
**Parent Task:** DEV-596 — Grafana Dashboard Creation
**Author:** Thanusha Bai V
**Date:** 2026-09-24

---

## **1. Overview**

This document is the research foundation for the Grafana dashboards that will be built in DEV-598, DEV-599, and DEV-600. It lists the Prometheus metrics exposed by HashiCorp Nomad, Consul, and Vault, and maps each metric to the specific dashboard panel where it will be used.

The goal of this research is to ensure that every PromQL query written in the dashboards references a real, documented metric — avoiding broken panels caused by guessing metric names.

---

## **2. Scope**

| Tool       | Dashboard Task | Purpose                                                            |
| ---------- | -------------- | ------------------------------------------------------------------ |
| **Nomad**  | DEV-598        | Cluster health, job allocations, resource utilization, deployments |
| **Consul** | DEV-599        | Service health, Raft leadership, network latency                   |
| **Vault**  | DEV-600        | Token usage, secret engine performance, audit logs                 |

---

## **3. How Metrics Are Exposed**

Each HashiCorp tool exposes Prometheus-format metrics via an HTTP endpoint on its agent:

| Tool       | Endpoint                              | Notes                                          |
| ---------- | ------------------------------------- | ---------------------------------------------- |
| **Nomad**  | `/v1/metrics?format=prometheus`       | Requires telemetry stanza configured           |
| **Consul** | `/v1/agent/metrics?format=prometheus` | Requires `prometheus_retention_time` set       |
| **Vault**  | `/v1/sys/metrics?format=prometheus`   | Requires a token with `read` on `/sys/metrics` |

### **Prerequisite Configurations**

* **Nomad:** `publish_allocation_metrics = true` and `publish_node_metrics = true` in the `telemetry` stanza.
* **Consul:** `prometheus_retention_time` must be set in the agent's `telemetry` config.
* **Vault:** Prometheus scraping must be enabled and authenticated with a valid Vault token.

---

# **4. Nomad Metrics (for DEV-598)**

## **4.1 Metric Reference Table**

| Metric Name                            | Type    | Description                                | Dashboard Panel     |
| -------------------------------------- | ------- | ------------------------------------------ | ------------------- |
| `nomad_client_host_cpu_total`          | Counter | Total CPU ticks used by the host           | CPU Utilization     |
| `nomad_client_host_cpu_idle`           | Counter | Idle CPU ticks on the host                 | CPU Utilization     |
| `nomad_client_host_memory_used`        | Gauge   | Memory used by the host (bytes)            | Memory Utilization  |
| `nomad_client_host_memory_total`       | Gauge   | Total memory available on the host (bytes) | Memory Utilization  |
| `nomad_client_host_disk_used`          | Gauge   | Disk space used by the host (bytes)        | Disk Utilization    |
| `nomad_client_host_disk_size`          | Gauge   | Total disk size on the host (bytes)        | Disk Utilization    |
| `nomad_client_allocated_cpu`           | Gauge   | CPU allocated to all jobs (MHz)            | CPU Allocation      |
| `nomad_client_unallocated_cpu`         | Gauge   | CPU not allocated to jobs (MHz)            | CPU Allocation      |
| `nomad_client_allocated_memory`        | Gauge   | Memory allocated to all jobs (bytes)       | Memory Allocation   |
| `nomad_client_unallocated_memory`      | Gauge   | Memory not allocated to jobs (bytes)       | Memory Allocation   |
| `nomad_client_allocs_cpu_total_ticks`  | Counter | CPU ticks used by allocations              | Actual CPU Usage    |
| `nomad_client_allocs_cpu_allocated`    | Gauge   | CPU allocated to allocations (MHz)         | Actual CPU Usage    |
| `nomad_client_allocs_memory_usage`     | Gauge   | Memory used by allocations (bytes)         | Actual Memory Usage |
| `nomad_client_allocs_memory_allocated` | Gauge   | Memory allocated to allocations (bytes)    | Actual Memory Usage |
| `nomad_nomad_job_summary_running`      | Gauge   | Jobs currently in running state            | Job Status          |
| `nomad_nomad_job_summary_pending`      | Gauge   | Jobs currently in pending state            | Job Status          |
| `nomad_nomad_job_summary_failed`       | Gauge   | Jobs currently in failed state             | Job Status          |
| `nomad_client_allocations_blocked`     | Gauge   | Number of blocked allocations              | Allocation Status   |
| `nomad_client_allocations_migrating`   | Gauge   | Number of migrating allocations            | Allocation Status   |

---

## **4.2 PromQL Queries**

### **CPU Utilization %**

```promql
sum(nomad_client_host_cpu_total) / (sum(nomad_client_host_cpu_total) + sum(nomad_client_host_cpu_idle))
```

### **Memory Utilization %**

```promql
sum(nomad_client_host_memory_used) / sum(nomad_client_host_memory_total)
```

### **Disk Utilization %**

```promql
sum(nomad_client_host_disk_used) / sum(nomad_client_host_disk_size)
```

### **CPU Allocated %**

```promql
sum(nomad_client_allocated_cpu) / (sum(nomad_client_allocated_cpu) + sum(nomad_client_unallocated_cpu))
```

### **Memory Allocated %**

```promql
sum(nomad_client_allocated_memory) / (sum(nomad_client_allocated_memory) + sum(nomad_client_unallocated_memory))
```

### **Job Status (Running / Pending / Failed)**

```promql
nomad_nomad_job_summary_running
nomad_nomad_job_summary_pending
nomad_nomad_job_summary_failed
```

---

# **5. Consul Metrics (for DEV-599)**

## **5.1 Metric Reference Table**

| Metric Name                       | Type    | Description                             | Dashboard Panel               |
| --------------------------------- | ------- | --------------------------------------- | ----------------------------- |
| `consul_raft_leader_lastContact`  | Gauge   | Duration since last contact with leader | Raft Leadership               |
| `consul_raft_commitTime`          | Gauge   | Time to commit Raft log entries         | Raft Health / Network Latency |
| `consul_raft_apply`               | Counter | Number of Raft log commits              | Raft Activity                 |
| `consul_raft_state_candidate`     | Counter | Raft state transitions to candidate     | Leader Elections              |
| `consul_raft_state_leader`        | Counter | Raft state transitions to leader        | Leader Elections              |
| `consul_autopilot_healthy`        | Gauge   | 1 if Autopilot is healthy, 0 otherwise  | Service Health Checks         |
| `consul_consul_members_servers`   | Gauge   | Number of Consul servers active         | Cluster Size                  |
| `consul_dns_domain_query_count`   | Counter | Number of DNS domain queries            | DNS Load                      |
| `consul_dns_domain_query`         | Summary | Time spent on DNS domain queries        | DNS Latency                   |
| `consul_kvs_apply_count`          | Counter | Number of KV store applies              | KV Load                       |
| `consul_catalog_register_count`   | Counter | Number of catalog register operations   | Service Discovery             |
| `consul_catalog_deregister_count` | Counter | Number of catalog deregister operations | Service Discovery             |
| `consul_acl_ResolveToken_count`   | Counter | Number of ACL token resolutions         | ACL Activity                  |
| `consul_raft_wal_log_appends`     | Counter | Number of log batches appended to WAL   | WAL Performance               |

---

## **5.2 PromQL Queries**

### **Raft Leader Last Contact**

```promql
consul_raft_leader_lastContact != 0
```

### **Raft Commits (per 5 minutes)**

```promql
rate(consul_raft_apply[5m])
```

### **Leader Election Events**

```promql
rate(consul_raft_state_candidate[1m])
rate(consul_raft_state_leader[1m])
```

### **Autopilot Health / Service Health Checks**

```promql
consul_autopilot_healthy
```

### **DNS Queries (per 5 minutes)**

```promql
rate(consul_dns_domain_query_count[5m])
```

### **Raft Commit Time (network latency indicator)**

```promql
consul_raft_commitTime
```

---

# **6. Vault Metrics (for DEV-600)**

## **6.1 Metric Reference Table**

| Metric Name                          | Type    | Description                                    | Dashboard Panel           |
| ------------------------------------ | ------- | ---------------------------------------------- | ------------------------- |
| `vault_token_count`                  | Gauge   | Number of tokens in the system                 | Token Usage               |
| `vault_token_creation`               | Counter | Number of tokens created                       | Token Usage               |
| `vault_expire_num_leases`            | Gauge   | Number of leases eligible for expiry           | Lease Management          |
| `vault_expire_lease_expiration`      | Summary | Time to expire leases                          | Lease Performance         |
| `vault_route_create_*`               | Counter | Route creation operations per mount            | Secret Engine Performance |
| `vault_route_read_*`                 | Counter | Route read operations per mount                | Secret Engine Performance |
| `vault_audit_log_request`            | Counter | Number of audit log requests                   | Audit Statistics          |
| `vault_audit_log_request_failure`    | Counter | Number of audit log failures                   | Audit Statistics          |
| `vault_audit_log_response`           | Counter | Number of audit log responses                  | Audit Statistics          |
| `vault_core_unsealed`                | Gauge   | 1 if Vault is unsealed, 0 if sealed            | Vault Status              |
| `vault_ha_rpc_client_forward`        | Summary | Time to forward request from standby to active | HA Performance            |
| `vault_ha_rpc_client_forward_errors` | Counter | Number of standby forwarding failures          | HA Errors                 |
| `vault_runtime_alloc_bytes`          | Gauge   | Allocated memory (bytes)                       | Resource Usage            |
| `vault_runtime_num_goroutines`       | Gauge   | Number of goroutines                           | Resource Usage            |

---

## **6.2 PromQL Queries**

### **Token Count**

```promql
vault_token_count
```

### **Token Creation Rate**

```promql
rate(vault_token_creation[5m])
```

### **Lease Expiration**

```promql
vault_expire_num_leases
```

### **Secret Engine Requests (per mount)**

```promql
rate(vault_route_create_*[5m])
rate(vault_route_read_*[5m])
```

### **Audit Log Failures**

```promql
rate(vault_audit_log_request_failure[5m])
```

### **Vault Seal Status**

```promql
vault_core_unsealed
```

### **HA Forward Time**

```promql
vault_ha_rpc_client_forward
```

---

# **7. Metric-to-Panel Mapping Summary**

| Task        | Dashboard Panel            | Metrics Used                                                             |
| ----------- | -------------------------- | ------------------------------------------------------------------------ |
| **DEV-598** | CPU Utilization            | `nomad_client_host_cpu_total`, `nomad_client_host_cpu_idle`              |
| **DEV-598** | Memory Utilization         | `nomad_client_host_memory_used`, `nomad_client_host_memory_total`        |
| **DEV-598** | Disk Utilization           | `nomad_client_host_disk_used`, `nomad_client_host_disk_size`             |
| **DEV-598** | Job Allocations & Failures | `nomad_nomad_job_summary_running/pending/failed`                         |
| **DEV-598** | Deployment Status          | `nomad_client_allocations_blocked`, `nomad_client_allocations_migrating` |
| **DEV-599** | Service Health Checks      | `consul_autopilot_healthy`                                               |
| **DEV-599** | Raft Leadership            | `consul_raft_leader_lastContact`, `consul_raft_state_leader`             |
| **DEV-599** | Network Latency            | `consul_raft_commitTime`, `consul_raft_apply`                            |
| **DEV-600** | Token Usage                | `vault_token_count`, `vault_token_creation`                              |
| **DEV-600** | Secret Engine Performance  | `vault_route_create_*`, `vault_route_read_*`                             |
| **DEV-600** | Audit Log Statistics       | `vault_audit_log_request`, `vault_audit_log_request_failure`             |

---

# **8. Notes for Dashboard Building**

1. **Nomad telemetry prerequisites:** Ensure `publish_allocation_metrics = true` and `publish_node_metrics = true` are set in the Nomad client's `telemetry` stanza, otherwise allocation-level metrics will not appear.
2. **Consul Prometheus retention:** `prometheus_retention_time` must be configured in the Consul agent's `telemetry` stanza for `/v1/agent/metrics?format=prometheus` to return data.
3. **Vault authentication:** Scraping Vault metrics via Prometheus requires a valid Vault token with `read` capability on `/sys/metrics`. This token is typically provided to Prometheus via a file-based secret or environment variable.
4. **Vault metric name transformation:** Forward slashes in Vault mount point paths are converted to dashes in metric names. For example, `auth/token` becomes `auth-token`, and `secret/data` becomes `secret-data`.
5. **Grafana Cloud trial limitation:** This dashboard is being developed on a Grafana Cloud trial instance. The exported JSON files are portable and can be imported into any Grafana instance (self-hosted, EC2, or Cloud).
6. **Data source requirements:** All three dashboards assume a Prometheus data source named `Prometheus` is configured in Grafana. The node metrics and log integration (parent task DEV-596) additionally require a Loki data source named `Loki`.

---

# **9. References**

* **Nomad Metrics Reference:** https://developer.hashicorp.com/nomad/docs/operations/metrics-reference
* **Consul Telemetry Documentation:** https://developer.hashicorp.com/consul/docs/agent/telemetry
* **Vault Telemetry Metrics:** https://developer.hashicorp.com/vault/docs/internals/telemetry/metrics
* **Grafana Prometheus Data Source:** https://grafana.com/docs/grafana/latest/datasources/prometheus/

---

# **End of Report**
