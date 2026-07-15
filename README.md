# Docker Monitoring Stack

> Full observability pipeline: containerized app deployment · metric collection · log aggregation · real-time dashboards · centralized diagnostics — orchestrated via Docker Compose.

[![Prometheus](https://img.shields.io/badge/Prometheus-Latest-E6522C?style=flat-square&logo=prometheus&logoColor=white)](#)
[![Grafana](https://img.shields.io/badge/Grafana-Latest-F46800?style=flat-square&logo=grafana&logoColor=white)](#)
[![Loki](https://img.shields.io/badge/Loki-2.9.8-2C3239?style=flat-square&logo=grafana&logoColor=white)](#)
[![Docker Compose](https://img.shields.io/badge/Docker_Compose-Orchestrated-2496ED?style=flat-square&logo=docker&logoColor=white)](#)
[![Platform](https://img.shields.io/badge/Platform-WSL2%20%2F%20Windows-0078D4?style=flat-square&logo=windows&logoColor=white)](#)
[![Status](https://img.shields.io/badge/Status-Core%20Pipeline%20Complete-brightgreen?style=flat-square)](#)
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)
[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-2.1-4baaaa?style=flat-square)](.github/CODE_OF_CONDUCT.md)
[![ISC2](https://img.shields.io/badge/Cert-ISC2%20CC-purple?style=flat-square)](#)

---

## Overview

This project documents the deployment and operation of a **production-grade observability stack** using open-source tools orchestrated via Docker Compose. The goal is to demonstrate a real-world monitoring pipeline — from application deployment to metric collection, log aggregation, real-time visualization, and centralized diagnostics — covering skills directly applicable to DevOps, SRE, and Cloud Security roles.

The monitored application is a **WordPress + MySQL** stack, chosen as a realistic web application scenario. On top of it, a full observability layer is deployed: **Prometheus** for metric storage, **cAdvisor** for container metric exposure, **Loki + Promtail** for centralized log management, and **Grafana** as the unified visualization frontend.

The core pipeline — deployment, visualization, and log correlation — is complete and fully reproducible. See [Roadmap](#roadmap) for planned extensions.

---

## Lab Environment

| Component | Details |
| --- | --- |
| Host OS | Windows + WSL2 (Ubuntu) |
| Container engine | Docker Desktop (WSL2 backend) + Docker Compose V2 |
| Application | WordPress + MySQL 8.0 |
| Metric collection | Prometheus + cAdvisor v0.47.0 |
| Log aggregation | Loki 2.9.8 + Promtail 2.9.8 (pinned for schema compatibility) |
| Visualization | Grafana (Dashboard ID 14282 — cAdvisor exporter) |
| Total containers | 7 (application + monitoring stack) |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Compose Network                  │
│                     (monitoring_network)                    │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐                         │
│  │  WordPress   │    │   MySQL     │   Application Layer    │
│  │   :8080      │───▶│   :3306     │                        │
│  └──────┬───────┘    └──────┬──────┘                        │
│         │                   │                               │
│  ═══════╪═══════════════════╪═════════════════════════════  │
│         │                   │                               │
│  ┌──────▼───────────────────▼──────┐                        │
│  │           cAdvisor :8081        │   Metric Exposure      │
│  │   (discovers all containers)    │                        │
│  └──────────────┬──────────────────┘                        │
│                 │  scrape /metrics                          │
│  ┌──────────────▼──────────────────┐                        │
│  │       Prometheus :9090          │   Metric Storage       │
│  │   (time-series database)        │                        │
│  └──────────────┬──────────────────┘                        │
│                 │                                           │
│  ┌──────────────▼──────────────────┐                        │
│  │         Grafana :3000           │   Visualization        │
│  │   (dashboards + log explorer)   │                        │
│  └──────────────▲──────────────────┘                        │
│                 │                                           │
│  ┌──────────────┴──────────────────┐                        │
│  │          Loki :3100             │   Log Storage          │
│  └──────────────▲──────────────────┘                        │
│                 │                                           │
│  ┌──────────────┴──────────────────┐                        │
│  │       Promtail (agent)          │   Log Collection       │
│  │   (Docker SD via socket)        │                        │
│  └─────────────────────────────────┘                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> A detailed architecture walkthrough is available in [`architecture/stack_architecture.md`](architecture/stack_architecture.md).

---

## Phases

| Phase | Title | Status |
| --- | --- | --- |
| [Architecture](architecture/stack_architecture.md) | Design stack architecture & component selection | ✅ Complete |
| [Phase 1](phases/phase-1-environment-setup.md) | Infrastructure Deployment & Zero-Instrumentation Telemetry | ✅ Complete |
| [Phase 2](phases/phase-2-telemetry-visualization.md) | Telemetry Visualization & WSL2 Architectural Constraints | ✅ Complete |
| [Phase 3](phases/phase-3-log-pipeline-correlation.md) | Log Pipeline Validation & Cross-Source Correlation | ✅ Complete |

> **Project status:** The core observability pipeline — deployment, visualization, and log correlation — is complete, tested, and fully reproducible via `docker compose up -d`. Alerting and production-hardening are captured as future work in the [Roadmap](#roadmap) below rather than tracked as in-progress phases.

---

## Key Objectives

**Single-command deployment** — The entire 7-container stack (application + monitoring) is deployed via a single `docker compose up -d` command. All services are pre-configured through mounted configuration files, requiring zero manual post-boot intervention.

**Three pillars of observability** — The stack implements the three pillars of modern observability: metrics (Prometheus + cAdvisor), logs (Loki + Promtail), and visualization (Grafana), unified under a single-pane-of-glass dashboard.

**Container-native metric collection** — cAdvisor automatically discovers every running container and exposes CPU, memory, filesystem, and network metrics in Prometheus-compatible format, requiring no application-level instrumentation.

**Centralized log aggregation** — Promtail harvests Docker container logs via Docker service discovery (through `/var/run/docker.sock`), labels them by container name, and ships them to Loki for indexed storage and ad-hoc querying through Grafana's Explore interface.

**WSL2-aware query engineering** — PromQL queries have been adapted to compensate for the cgroup namespace constraints imposed by Docker Desktop on WSL2, demonstrating the ability to deliver reliable telemetry across non-trivial virtualization boundaries.

**Cross-source incident correlation** — A synthetic attack scenario (forced 404 enumeration) demonstrates the full SOC diagnostic loop: detecting a metric anomaly in Prometheus, pinpointing the exact time window, and attributing root cause via correlated Loki logs — reducing diagnosis from minutes to seconds.

**Threat-hunting query arsenal** — A reusable set of PromQL and LogQL queries covers common threat scenarios: network exfiltration spikes, compute saturation (cryptojacking), memory exhaustion, and container churn (crash loops / persistence attempts).

---

## Stack Components

| Service | Image | Port | Role |
| --- | --- | --- | --- |
| WordPress | `wordpress:latest` | 8080 | Web application (monitored target) |
| MySQL | `mysql:8.0` | internal | Database backend (no external port) |
| Prometheus | `prom/prometheus:latest` | 9090 | Time-series metric database |
| Grafana | `grafana/grafana:latest` | 3000 | Visualization & dashboards |
| Loki | `grafana/loki:2.9.8` | 3100 | Log aggregation engine |
| Promtail | `grafana/promtail:2.9.8` | — | Log collection agent |
| cAdvisor | `gcr.io/cadvisor/cadvisor:v0.47.0` | 8081 | Container metric exporter |

---

## Notable Technical Challenges Solved

**Loki 3.x schema deprecation** — The `grafana/loki:latest` tag pulled the 3.x generation, which enforces `tsdb` index type and `schema v13`, breaking the stable `boltdb-shipper` configuration and causing an immediate container exit. Resolved by pinning both Loki and Promtail to `2.9.8`, the last stable release of the previous generation.

**UID/GID permission collision on persistent volumes** — The Loki service entered a crash loop with `mkdir /tmp/loki/rules: permission denied`. Root cause: Docker provisioned the `loki_data` volume with host-level `root` ownership, while the official Loki image runs as an unprivileged internal UID. Resolved by injecting `user: "root"` into the Loki service definition, aligning the container's execution context with the volume's ownership.

**WSL2 clock drift breaking Prometheus queries** — Prometheus reported 38-second out-of-sync warnings after Windows host sleep states, producing empty Grafana queries due to future-timestamp lookups. Resolved by restarting the Docker Engine from the Windows host, forcing the WSL2 VM to resynchronize its hardware clock (`hwclock`) with the host motherboard.

**cAdvisor cgroup blindness on WSL2** — WSL2 encapsulates the Docker daemon inside a lightweight VM, preventing cAdvisor from enumerating individual container cgroups. Per-container labels (`Host`, `Container name`) failed to populate. Resolved by shifting from micro-monitoring to a macro-monitoring strategy, rewriting PromQL queries to target the aggregated root cgroup (`id="/docker"` for CPU/memory, `id="/"` for network) with `sum()` aggregations.

**Cross-source attack attribution** — During a synthetic 404-enumeration attack, isolated metrics alone (an RX spike) provided a symptom but no attribution. Resolved by pairing the Prometheus spike with a time-boxed LogQL query against Loki, using split-screen Explore to identify the exact requesting URI and source IP within the same millisecond window.

---

## Repository Structure

```
docker-monitoring-stack/
│
├── LICENSE
├── README.md
├── .gitignore
├── docker-compose.yml
│
├── .github/
│   ├── CODE_OF_CONDUCT.md
│   ├── CONTRIBUTING.md
│   ├── SECURITY.md
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── ISSUE_TEMPLATE/
│       ├── config.yml
│       ├── bug-report.yml
│       └── feature-request.yml
│
├── architecture/
│   └── stack_architecture.md
│
├── config/
│   ├── prometheus/
│   │   └── prometheus.yml
│   ├── loki/
│   │   └── loki-config.yml
│   └── promtail/
│       └── promtail-config.yml
│
└── phases/
    ├── phase-1-environment-setup.md
    ├── phase-2-telemetry-visualization.md
    └── phase-3-log-pipeline-correlation.md
```

---

## Roadmap

Future work, not currently tracked as in-progress phases:

- **Alert engineering** — Prometheus alert rules (`alert.rules.yml`) with notification channels (Slack/email webhooks), building directly on the PromQL threat-hunting queries from Phase 3.
- **Stack hardening** — Resource limits, non-default Grafana auth, HTTPS reverse proxy, `.env` parameterization for secrets currently hardcoded in `docker-compose.yml`.
- **Distributed tracing** — Integrate Tempo to complete the third pillar of observability alongside metrics and logs.
- **Host-level metrics** — Add Node Exporter for host telemetry beyond container scope.

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/alejandroZ345/docker-monitoring-stack.git
cd docker-monitoring-stack

# Deploy the full stack
docker compose up -d

# Verify all 7 containers are running
docker compose ps

# Access the services
# Grafana:    http://localhost:3000  (admin / admin)
# Prometheus: http://localhost:9090
# WordPress:  http://localhost:8080
# cAdvisor:   http://localhost:8081
```

> For the full setup walkthrough, start with [Phase 1](phases/phase-1-environment-setup.md).

---

## Contributing

Contributions are welcome — bug reports, configuration fixes, documentation improvements, or suggestions tied to the [Roadmap](#roadmap) above. Please read the [Contributing Guide](.github/CONTRIBUTING.md) and [Code of Conduct](.github/CODE_OF_CONDUCT.md) before opening an issue or pull request. Security issues should be reported privately per the [Security Policy](.github/SECURITY.md) rather than as a public issue.

---

## About

**Alejandro Zavala** — Systems Engineer & Cybersecurity Professional
ISC2 Certified in Cybersecurity (CC) · Specialization: SOC Operations, Infrastructure Monitoring, Technical Documentation
[linkedin.com/in/alejandro-zavala-zenteno](https://www.linkedin.com/in/alejandro-zavala-zenteno) · [ISC2 Badge](https://www.credly.com/badges/fe6eeefe-d684-45fa-8f57-9f7d025db2d6/public_url)

Related project: [wazuh-soc-homelab](https://github.com/alejandroZ345/wazuh-soc-homelab) — enterprise SIEM/XDR platform with Wazuh + TheHive.
