# Stack Architecture

> Design rationale, component selection, and data flow for the Docker Monitoring Stack.

## Design Goals

1. **Zero-instrumentation monitoring** — The application (WordPress + MySQL) requires no code changes or sidecars. All telemetry is collected externally.
2. **Single-command deployment** — Every service, including its configuration, is defined in `docker-compose.yml`.
3. **Unified interface** — All metrics and logs converge in Grafana, eliminating context-switching between tools.
4. **Virtualization-aware telemetry** — Query design accounts for WSL2 cgroup abstractions, preserving observability across non-trivial host boundaries.

## Component Selection

| Component | Why this tool? | Alternatives considered |
| --- | --- | --- |
| **Prometheus** | Industry standard for cloud-native metrics. Pull-based model is firewall-friendly. | InfluxDB (push model, requires agents) |
| **cAdvisor** | Auto-discovers containers, zero config. Exposes Prometheus-native metrics. | Docker stats API (no persistence) |
| **Loki 2.9.8** | Label-indexed log storage — lightweight, cost-effective. Designed for Grafana. Pinned to 2.x to retain `boltdb-shipper` compatibility. | ELK Stack (heavier, full-text indexing); Loki 3.x (breaks local storage schema) |
| **Promtail 2.9.8** | Native Loki agent. Uses Docker service discovery via the host socket. | Fluentd/Fluent Bit (more complex config) |
| **Grafana** | Multi-datasource dashboards. Native support for Prometheus + Loki. | Kibana (Elasticsearch-only) |

## Data Flow

### Metrics Pipeline

```
Container → cAdvisor (auto-discovery) → /metrics endpoint → Prometheus (scrape every 15s) → Grafana (query + visualize)
```

### Logs Pipeline

```
Container → Docker daemon → Promtail (Docker SD via socket) → Loki (store + label-index) → Grafana Explore (LogQL query)
```

## Network Topology

All services communicate through a shared Docker Compose network (`monitoring_network`). External access is exposed only through mapped ports on `localhost`:

| Service | Internal address | External port |
| --- | --- | --- |
| WordPress | `wordpress:80` | `localhost:8080` |
| MySQL | `mysql:3306` | Not exposed |
| Prometheus | `prometheus:9090` | `localhost:9090` |
| Grafana | `grafana:3000` | `localhost:3000` |
| Loki | `loki:3100` | `localhost:3100` |
| cAdvisor | `cadvisor:8080` | `localhost:8081` |
| Promtail | `promtail` | Not exposed |

## WSL2 Considerations

Running Docker Desktop over WSL2 introduces virtualization abstractions that affect cgroup visibility:

- **cgroup namespace isolation** — cAdvisor cannot enumerate individual container cgroups. PromQL queries target the aggregated root (`id="/docker"`) instead of per-container labels, producing Docker-Engine-wide metrics rather than per-container breakdowns.
- **Single virtualized network switch** — All container network I/O aggregates at the VM root interface. Network queries use `id="/"` with `sum()` aggregation to consolidate traffic across `eth0` and `lo`.
- **Clock drift after host sleep** — The WSL2 utility VM freezes its clock during Windows sleep states. A Docker Engine restart from Windows resynchronizes the VM's hardware clock.

See [Phase 2](../phases/phase-2-telemetry-visualization.md) for the full engineering log on these constraints.

## Security Considerations (Lab Context)

- Grafana runs with default credentials (`admin/admin`) — acceptable for a local lab, documented for hardening in a later phase.
- cAdvisor mounts `/var/run/docker.sock` as read-only — necessary for container discovery but grants metadata visibility.
- Loki runs as `user: "root"` to resolve a UID/GID collision on the persistent volume — a lab-only compromise. Production should use an `initContainer` to `chown` the volume to Loki's unprivileged UID.
- No inter-service TLS — all traffic is internal to the Docker network.
- MySQL credentials are hardcoded in `docker-compose.yml` — a later hardening phase migrates these to `.env` files.
