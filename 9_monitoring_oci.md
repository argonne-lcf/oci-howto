## Monitoring

The **super-ant** cluster includes a dedicated monitoring node, **`super-ant-monitoring`** (`172.16.0.106`), which runs:

- **Prometheus** – time-series metrics collection and storage  
- **Grafana** – interactive dashboards and visualizations on top of Prometheus (and other data sources)

This section shows how to access the Grafana UI and what each of the pre-configured dashboards provides.

---

### 1. Accessing Grafana

Grafana for the super-ant cluster is exposed via the following URL:

- <https://grafana.138.2.228.227.endpoint.oci-hpc.ai/>

Default credentials (change in production environments):

- **Username:** `admin`  
- **Password:** `WQ32gthVAWb-t9jg`  

After your first login, you should:

1. Go to **Configuration → Users** and change the `admin` password.
2. Optionally create additional read-only or per-team user accounts.
3. Confirm that the Prometheus data source is healthy under **Configuration → Data sources**.

---

### 2. GPU Metrics Dashboards

Several dashboards are focused on GPU health and utilization across the H200 nodes.

#### 2.1 GPU Metrics 1 – Overview

This dashboard provides a high-level overview of GPU usage, including:

- Per-GPU utilization (%)  
- Memory usage (GB)  
- Power draw and temperature  
- Aggregated stats per node

<img src="images/9_monitoring/1.png" alt="Grafana GPU Metrics 1 dashboard - overall GPU utilization and memory usage" width="75%">

#### 2.2 GPU Metrics 2 – Per-GPU detail

This dashboard drills down into individual GPUs:

- Per-GPU SM utilization  
- Memory bandwidth and PCIe traffic  
- ECC error counters (where available)

<img src="images/9_monitoring/2.png" alt="Grafana GPU Metrics 2 dashboard - detailed per-GPU metrics" width="75%">

#### 2.3 GPU Status and Health

This dashboard focuses on:

- GPU health flags  
- ECC status and XID error counters  
- Throttling / clock capping indicators

<img src="images/9_monitoring/3.png" alt="Grafana GPU status dashboard - GPU health and ECC/XID status" width="75%">

#### 2.4 GPU Metrics 3 – Performance/throughput focused

This dashboard is tuned for:

- GPU compute throughput metrics  
- Data transfer rates (host↔device, device↔device)  
- Correlation between utilization, memory usage, and power

<img src="images/9_monitoring/8.png" alt="Grafana GPU Metrics 3 dashboard - GPU throughput and performance metrics" width="75%">

---

### 3. File Storage and Host Metrics

#### 3.1 File Storage Status

This dashboard tracks filesystem usage and I/O stats, including:

- Capacity and utilization for `/config`, `/home`, `/fss`, and local disks  
- Read/write throughput and IOPS over time  
- Potential bottlenecks or near-full volumes

<img src="images/9_monitoring/4.png" alt="Grafana File Storage Status dashboard - filesystem capacity and I/O metrics" width="75%">

#### 3.2 Host Metrics

This dashboard summarizes host-level metrics:

- CPU load, core utilization, and run queue length  
- System memory usage and swap activity  
- Network throughput and error counters per interface

<img src="images/9_monitoring/10.png" alt="Grafana Host Metrics dashboard - CPU, memory, and network metrics" width="75%">

---

### 4. Multinode and Cluster-wide Metrics

#### 4.1 Multinode Metrics

The multinode dashboard aggregates metrics across the cluster to help you:

- Compare GPU and CPU utilization across nodes  
- Detect stragglers or imbalanced workloads  
- Visualize overall cluster health at a glance

<img src="images/9_monitoring/5.png" alt="Grafana Multinode Metrics dashboard - cluster-wide GPU and CPU metrics" width="75%">

#### 4.2 All Metrics (exploratory view)

This dashboard provides a broad, “kitchen sink” view of many metrics:

- Combined GPU, CPU, memory, and network views  
- Useful for exploratory debugging and correlation across subsystems

<img src="images/9_monitoring/6.png" alt="Grafana All Metrics dashboard - combined system metrics" width="75%">

---

### 5. CPU and Frequency Metrics

#### 5.1 CPU Metrics

This dashboard focuses specifically on CPU behavior:

- Per-core utilization and load  
- Context switches and interrupts  
- Correlation between CPU load and GPU workload

<img src="images/9_monitoring/7.png" alt="Grafana CPU Metrics dashboard - per-core CPU performance" width="75%">

#### 5.2 Frequency Metrics

This dashboard tracks frequency scaling and throttling:

- CPU and GPU clock frequencies over time  
- Thermal throttling or power-capping events  
- Relationship between frequency, utilization, and temperature

<img src="images/9_monitoring/9.png" alt="Grafana Frequency Metrics dashboard - CPU/GPU clock frequencies and throttling" width="75%">

---

### 6. Billing and Cost Metrics

The monitoring stack also includes dashboards that visualize OCI billing and cost metrics for the Argonne-AI tenancy.

#### 6.1 Billing 1 – High-level cost overview

This dashboard provides:

- Total cost over time  
- Breakdown by compartment, shape, or service  
- Quick view of daily or monthly cost trends

<img src="images/9_monitoring/11.png" alt="Grafana Billing 1 dashboard - high-level cost overview" width="75%">

#### 6.2 Billing 2 – Detailed cost breakdown

This dashboard gives a more detailed breakdown:

- Per-project or per-compartment charts  
- Cost attribution by resource type (compute, storage, networking)  
- Filters to isolate specific time ranges or services

<img src="images/9_monitoring/12.png" alt="Grafana Billing 2 dashboard - detailed cost and compartment breakdown" width="75%">

---

With these dashboards, you can:

- Monitor **GPU utilization and health** across H200 nodes.  
- Track **CPU, memory, storage, and network** behavior.  
- Correlate **performance and resource usage** with **billing and cost**.  

This provides a complete view of both **technical health** and **operational cost** for the super-ant cluster.
