# Phase 3 — Log Pipeline Validation & Cross-Source Correlation

> Validate the end-to-end log aggregation pipeline (Promtail → Loki → Grafana), demonstrate advanced incident response capabilities through cross-source correlation, and engineer a PromQL/LogQL query arsenal for threat hunting.

---

## 1. Objective

Prove the stack's ability to unify metric anomalies (symptoms) with application logs (root causes) in a single chronological timeline — the foundational workflow of any modern SOC. This phase elevates the deployment from a passive monitoring platform to a tactical incident response workstation.

By the end of this phase:

- The log pipeline (Promtail → Loki → Grafana) is validated end-to-end.
- A synthetic attack scenario demonstrates cross-source correlation between metrics and logs.
- A reusable PromQL query arsenal is engineered for common threat hunting scenarios.
- LogQL's log-to-metric transformation is operational, bridging log storage and future automated alerting.

---

## 2. Visual Log Exploration (The Grafana Builder)

Grafana Explore provides two modes for LogQL (Loki's query language): **Code** (raw text) and **Builder** (interactive UI). For rapid triage, the Builder mode allows analysts to construct complex filters without memorizing syntax — a critical advantage during high-pressure incident response.

| Builder Component | LogQL Equivalent | Purpose |
| --- | --- | --- |
| **Label Filters** | `{container="wordpress"}` | Primary stream selector. Loki discards all non-matching streams before any processing, drastically reducing query time. |
| **Line Contains** | `\|= "error"` | Pipeline filter searching for specific strings within the already-filtered stream. |
| **Operations** | Chained `\|=` / `\|~` expressions | Combines multiple conditions. During validation, chained "Line contains" operations successfully isolated specific HTTP response codes (`404`) and severity levels (`error`). |

> **Key insight:** Label filters run before pipeline filters. Always constrain the stream with labels first — filtering 1 million lines is orders of magnitude slower than filtering 1,000.

---

## 3. Cross-Source Correlation (The SOC Workflow)

The true power of this observability stack lies in its forensic potential. During validation, a **synthetic attack** (forced `404` errors against a non-existent admin panel) was executed to demonstrate the correlation workflow.

Using Grafana's split-screen Explore feature, perfect chronological correlation was achieved:

| Panel | Data Source | Role in Investigation |
| --- | --- | --- |
| **Right — Metrics** | Prometheus | Detected a sudden spike in `Network Receive Bytes (RX)` — the **Indicator of Compromise (IoC) / Symptom**. |
| **Left — Logs** | Loki | Isolated the exact HTTP `GET` requests generating that traffic in the same millisecond — the **Root Cause / Attribution** (target URI + source IP). |

### The diagnostic loop

```
  Metric anomaly (Prometheus)
            │
            ▼
  Pinpoint time window
            │
            ▼
  Query logs in Loki (same window)
            │
            ▼
  Identify root cause / attacker
            │
            ▼
  Determine scope + remediation
```

> **Why this matters:** Traditional `docker logs` and `docker stats` show instantaneous, isolated data. This stack reduces the diagnostic window from minutes (or hours) of manual correlation to seconds of visual alignment.

---

## 4. PromQL Arsenal for Threat Hunting

To maximize the diagnostic value of Prometheus, the following queries have been engineered and tested against this architecture. Each query is mapped to a specific threat scenario, with function properties broken down for tactical reuse.

### 4.1 — Global Network Spikes (DDoS / Exfiltration)

```promql
sum(rate(container_network_receive_bytes_total{id="/"}[1m]))
```

- `sum()` — Aggregates all virtual interfaces (remember the WSL2 constraint from [Phase 2](phase-2-telemetry-visualization.md): all container traffic converges at the root cgroup).
- `rate(...[1m])` — Calculates the per-second average over a 1-minute sliding window. Converts absolute counters into readable throughput.
- **Tactical use:** Spot inbound flood patterns (DDoS) or sustained outbound transfers (data exfiltration).

### 4.2 — Compute Saturation (Cryptojacking)

```promql
rate(container_cpu_usage_seconds_total{id="/docker"}[5m])
```

- `[5m]` — A 5-minute window smooths out micro-spikes that would otherwise hide malicious behavior inside normal variance.
- **Tactical use:** Detects sustained CPU abuse (e.g., cryptomining processes running during idle hours) that would be invisible with 1-minute windows.

### 4.3 — OOM Risk (Memory Leaks / DoS)

```promql
container_memory_usage_bytes{id="/docker"}
```

- Absolute memory counter — no aggregation function needed.
- **Tactical use:** Baseline threshold alerting. A gradual upward trend over hours indicates a memory leak; a sudden ceiling approach indicates a memory-exhaustion DoS attempt.

### 4.4 — Container Churn (Crash Loops)

```promql
changes(container_start_time_seconds{id="/docker"}[15m])
```

- `changes()` — Counts how many times a value changed within the window.
- **Tactical use:** If a container's start time changes multiple times in 15 minutes, it indicates either a severe crash loop (bug) or an attacker repeatedly resetting the service (persistence attempt or DoS).

---

## 5. Advanced LogQL: Log-to-Metric Transformation

Beyond simple text searches, Loki's real value in a SOC context is its ability to **parse unstructured logs in real-time and convert them into time-series metrics**. This transformation is the critical bridge between log storage and automated alerting (to be implemented in Phase 5).

During validation, the target was the standard Apache "Combined Log Format" produced by WordPress. LogQL's `pattern` function proved to be the most efficient extraction method, avoiding the high CPU overhead associated with complex regular expressions.

### 5.1 — HTTP Status Code Extraction

```logql
sum by (status_code) (
  rate({container="wordpress"}
    | pattern `<ip> - - [<time>] "<method> <uri> <protocol>" <status_code> <size> <_>`
  [$__auto])
)
```

**Breakdown:**

- `{container="wordpress"}` — Label filter (fast stream selection).
- `| pattern ...` — Extracts named fields from each log line into LogQL labels.
- `<_>` — Discards the rest of the line (User-Agent, Referer), saving memory during the query.
- `[$__auto]` — Grafana variable that adapts the rate window to the current dashboard zoom level.
- `sum by (status_code)` — Groups the resulting rate by HTTP status, producing a time-series per code (`200`, `404`, `500`, etc.).

**Tactical use:** Visualize the ratio of legitimate traffic (`2xx`) vs. errors (`4xx`/`5xx`) over time. A sudden spike in `404` responses — as demonstrated in the synthetic attack — is a textbook enumeration / directory brute-force signature.

### 5.2 — Keyword-Based Error Rate

```logql
rate({container="wordpress"} |= "error" [$__auto])
```

- `|= "error"` — Pipeline filter matching any line containing the substring `error`.
- **Tactical use:** Generic error-rate baseline, useful for detecting anomalous bursts of any severity level without committing to a specific parser.

---

## 6. Final Validation

The end-to-end observability pipeline is fully operational. The integration of Prometheus and Loki via Grafana transforms isolated data points into actionable security intelligence, completing the shift from zero-instrumentation deployment to a **fully functional, tactical monitoring workstation**.

### Verification checklist

- [x] Promtail successfully discovers containers via Docker SD and ships logs to Loki
- [x] Grafana Explore queries Loki with both Builder and Code modes
- [x] Split-screen correlation between Prometheus metrics and Loki logs works in the same time window
- [x] Synthetic `404` attack was detected simultaneously in RX metrics and WordPress logs
- [x] All four PromQL threat-hunting queries return expected results against the running stack
- [x] LogQL `pattern` extraction successfully parses Apache Combined Log Format into labeled series

---

## Next Steps

With cross-source correlation validated, [Phase 4](./phase-4-performance-analysis.md) will formalize the diagnostic methodology uncovered here into a reusable incident response playbook: documenting the exact step-by-step workflow for common scenarios (high CPU, memory exhaustion, suspicious traffic spikes), including the specific PromQL + LogQL query pair used at each step. This converts tacit investigative knowledge into a repeatable SOC artifact.
