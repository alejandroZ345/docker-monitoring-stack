# Phase 2 — Telemetry Visualization & WSL2 Architectural Constraints

> Establish a centralized observability dashboard using Grafana, connect Prometheus as the primary time-series data source, and engineer around the virtualization boundaries imposed by running Docker Desktop over WSL2.

---

## 1. Objective

Activate the visualization tier of the observability stack deployed in [Phase 1](phase-1-environment-setup.md) by connecting Grafana to Prometheus and surfacing resource utilization (CPU, RAM, Network) of the Docker host and its active containers.

By the end of this phase:

- Grafana consumes Prometheus as a native data source via internal Docker DNS.
- A cAdvisor reference dashboard renders real-time CPU, memory, and network telemetry.
- All WSL2-specific virtualization quirks affecting metric collection are identified, documented, and resolved.

---

## 2. Implementation Strategy

| Step | Action |
| --- | --- |
| 1 | Provision the Grafana instance via Docker Compose (already defined in Phase 1). |
| 2 | Configure Prometheus as a Grafana data source using internal DNS: `http://prometheus:9090`. |
| 3 | Import the community cAdvisor dashboard (**ID: 14282**) as a visualization baseline. |
| 4 | Adjust PromQL queries to compensate for WSL2 architectural constraints (see [Section 3](#3-engineering-log-navigating-wsl2-virtualization-challenges)). |

---

## 3. Engineering Log: Navigating WSL2 Virtualization Challenges

During dashboard deployment, critical data-fetching failures were observed. Root Cause Analysis (RCA) revealed that running Docker Desktop over the Windows Subsystem for Linux (WSL2) introduces strict architectural abstractions not present in native Linux servers. Each constraint was diagnosed and resolved independently.

### Constraint 1 — WSL2 Time Drift (Clock Desynchronization)

**Issue:** Prometheus displayed critical out-of-sync warnings (e.g., a 38-second drift), resulting in `No data` queries in Grafana due to future-timestamp lookups.

**Analysis:** When the Windows host enters sleep or hibernation states, the WSL2 utility VM halts execution. Upon resumption, the VM's internal clock remains frozen relative to real time, and the Prometheus TSDB begins writing samples with stale timestamps. Grafana queries for "the last 5 minutes" then return empty result sets because the ingested data points appear to be in the future relative to the query window.

**Resolution:** Restarted the Docker Engine directly from the Windows host, forcing the WSL2 VM to resynchronize its hardware clock (`hwclock`) with the host motherboard. This immediately realigned Prometheus timestamps with real time and restored Grafana visibility.

> **Lesson learned:** Time synchronization is a non-negotiable prerequisite for any time-series monitoring stack. On WSL2, a Docker Engine restart after host sleep should be a standard recovery step.

---

### Constraint 2 — cAdvisor cgroup Blindness (Compute/Memory Metadata)

**Issue:** Container-specific dashboard variables (`Host`, `Container name`) failed to populate, breaking all granular PromQL queries that depended on per-container labels.

**Analysis:** WSL2 encapsulates the Docker daemon inside a lightweight virtual machine. cAdvisor, which relies on direct access to the kernel's cgroup hierarchy to enumerate individual containers, lacks the namespace privileges to cross this virtualization boundary. As a result, individual container labels (`container_label_io_kubernetes_container_name`, `name`) are not exposed to Prometheus — only aggregate cgroup identifiers are visible.

**Resolution:** Shifted from a **Micro-Monitoring** strategy (per-container metrics) to a **Macro-Monitoring** strategy (Docker Engine-wide metrics). PromQL queries were rewritten to target the aggregated root group `id="/docker"`, which surfaces the overall health of the Docker Engine rather than individual containers.

**Example query:**

```promql
rate(container_cpu_usage_seconds_total{id="/docker"}[1m])
```

> **Lesson learned:** When virtualization strips granularity, change the question — don't fight the architecture. The Docker Engine aggregate is still a meaningful signal for lab-scale monitoring.

---

### Constraint 3 — Network Interface Virtualization (Zero TX/RX Data)

**Issue:** Network IO panels displayed `0 B/s` for all containers, despite CPU and memory queries functioning correctly under the `/docker` cgroup.

**Analysis:** Unlike native Linux — which creates one virtual network interface (`vethX`) per container cgroup — WSL2 routes all container traffic through a **single virtualized network switch** at the utility VM level. As a consequence, network I/O counters never populate under the `/docker` cgroup and instead aggregate at the absolute root (`id="/"`), where all virtual interfaces (`eth0`, `lo`) are summed together.

**Resolution:** Re-engineered the PromQL network queries to target the absolute root (`id="/"`) and wrap them in a `sum()` aggregation function to consolidate all virtual interfaces into a single Docker Engine throughput metric.

**Applied queries:**

```promql
# Received traffic (RX)
sum(rate(container_network_receive_bytes_total{id="/"}[1m]))

# Transmitted traffic (TX)
sum(rate(container_network_transmit_bytes_total{id="/"}[1m]))
```

> **Lesson learned:** Always verify which cgroup level actually holds the counter you need. A query that works on bare-metal Linux can silently return zero under virtualized environments — and silent zeros are far more dangerous than explicit errors.

---

## 4. Final Validation

The dashboard was successfully stabilized. While individual container granularity is constrained by the Windows development environment, the adjusted Macro-Monitoring strategy provides a highly reliable, real-time baseline of the entire Docker Engine's performance.

### Verification checklist

- [x] Grafana connects to Prometheus via `http://prometheus:9090` without authentication errors
- [x] Dashboard 14282 renders CPU, Memory, and Network panels with live data
- [x] Time synchronization is stable after Docker Engine restart
- [x] PromQL queries return non-empty result sets across all panels
- [x] RX/TX network traffic panels display realistic throughput values

### Dashboard summary

| Panel category | Metric source | Strategy |
| --- | --- | --- |
| CPU Usage | `container_cpu_usage_seconds_total{id="/docker"}` | Macro (Docker Engine aggregate) |
| Memory Usage / Cached | `container_memory_*` on `/docker` cgroup | Macro (Docker Engine aggregate) |
| Received Network Traffic | `container_network_receive_bytes_total{id="/"}` | Root-level sum aggregation |
| Sent Network Traffic | `container_network_transmit_bytes_total{id="/"}` | Root-level sum aggregation |

---

## Next Steps

With visualization stabilized and the virtualization constraints fully documented, [Phase 3](phase-3-docker-compose.md) will focus on extending the pipeline toward log observability: validating Promtail's Docker service discovery, confirming Loki ingestion, and building the first cross-source queries that correlate metrics with logs inside Grafana's Explore interface.
