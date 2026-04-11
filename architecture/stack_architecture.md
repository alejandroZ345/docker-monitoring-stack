# Stack Architecture

> Design rationale, component selection, and data flow for the Docker Monitoring Stack.

## Design Goals

1. **Zero-instrumentation monitoring** — The application (WordPress + MySQL) requires no code changes or sidecars. All telemetry is collected externally.
2. **Single-command deployment** — Every service, including its configuration, is defined in `docker-compose.yml`.
3. **Unified interface** — All metrics and logs converge in Grafana, eliminating context-switching between tools.

## Component Selection

| Component | Why this tool? | Alternatives considered |
| --- | --- | --- |
| **Prometheus** | Industry standard for cloud-native metrics. Pull-based model is firewall-friendly. | InfluxDB (push model, requires agents) |
| **cAdvisor** | Auto-discovers containers, zero config. Exposes Prometheus-native metrics. | Docker stats API (no persistence) |
| **Loki** | Label-indexed log storage — lightweight, cost-effective. Designed for Grafana. | ELK Stack (heavier, full-text indexing) |
| **Promtail** | Native Loki agent. Reads Docker JSON log files directly from host. | Fluentd/Fluent Bit (more complex config) |
| **Grafana** | Multi-datasource dashboards. Native support for Prometheus + Loki. | Kibana (Elasticsearch-only) |

## Data Flow

### Metrics Pipeline

```
Container → cAdvisor (auto-discovery) → /metrics endpoint → Prometheus (scrape every 15s) → Grafana (query + visualize)
```

### Logs Pipeline

```
Container → Docker JSON log files → Promtail (tail + label) → Loki (store + index) → Grafana Explore (query)
```

## Network Topology

All services communicate through a shared Docker Compose network. External access is exposed only through mapped ports on `localhost`:

| Service | Internal address | External port |
| --- | --- | --- |
| WordPress | `wordpress:80` | `localhost:8080` |
| MySQL | `mysql:3306` | Not exposed |
| Prometheus | `prometheus:9090` | `localhost:9090` |
| Grafana | `grafana:3000` | `localhost:3000` |
| Loki | `loki:3100` | `localhost:3100` |
| cAdvisor | `cadvisor:8080` | `localhost:8081` |
| Promtail | `promtail` | Not exposed |

## Security Considerations (Lab Context)

- Grafana runs with default credentials (`admin/admin`) — acceptable for a local lab, documented for hardening in Phase 7.
- cAdvisor mounts `/var/run/docker.sock` as read-only — necessary for container discovery but grants metadata visibility.
- No inter-service TLS — all traffic is internal to the Docker network.
- MySQL credentials are hardcoded in `docker-compose.yml` — Phase 7 migrates these to `.env` files.
