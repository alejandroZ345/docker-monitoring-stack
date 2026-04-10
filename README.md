# Docker Monitoring Stack

> Full observability pipeline: containerized app deployment · metric collection · log aggregation · real-time dashboards · centralized diagnostics — orchestrated via Docker Compose.

[![Prometheus](https://img.shields.io/badge/Prometheus-Latest-E6522C?style=flat-square&logo=prometheus&logoColor=white)](#)
[![Grafana](https://img.shields.io/badge/Grafana-Latest-F46800?style=flat-square&logo=grafana&logoColor=white)](#)
[![Loki](https://img.shields.io/badge/Loki-Latest-2C3239?style=flat-square&logo=grafana&logoColor=white)](#)
[![Docker Compose](https://img.shields.io/badge/Docker_Compose-Orchestrated-2496ED?style=flat-square&logo=docker&logoColor=white)](#)
[![Status](https://img.shields.io/badge/Status-Active%20development-green?style=flat-square)](#)
[![ISC2](https://img.shields.io/badge/Cert-ISC2%20CC-purple?style=flat-square)](#)

---

## Overview

This project documents the deployment and operation of a **production-grade observability stack** using open-source tools orchestrated via Docker Compose. The goal is to demonstrate a real-world monitoring pipeline — from application deployment to metric collection, log aggregation, real-time visualization, and centralized diagnostics — covering skills directly applicable to DevOps, SRE, and Cloud Security roles.

The monitored application is a **WordPress + MySQL** stack, chosen as a realistic web application scenario. On top of it, a full observability layer is deployed: **Prometheus** for metric storage, **cAdvisor** for container metric exposure, **Loki + Promtail** for centralized log management, and **Grafana** as the unified visualization frontend.

The lab is structured as a series of documented phases, each building on the previous one, so that every configuration decision and troubleshooting step is reproducible and auditable.

---

## Lab Environment

| Component | Details |
| --- | --- |
| Host OS | Ubuntu 24.04 LTS (VMware Virtual Platform) |
| Container engine | Docker + Docker Compose V2 |
| Application | WordPress + MySQL 5.7 |
| Metric collection | Prometheus + cAdvisor (Google) |
| Log aggregation | Loki + Promtail (Grafana Labs) |
| Visualization | Grafana (Dashboard ID 193) |
| Total containers | 7 (application + monitoring stack) |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Compose Network                  │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐                         │
│  │  WordPress   │    │   MySQL     │   Application Layer     │
│  │   :8080      │───▶│   :3306     │                         │
│  └──────┬───────┘    └──────┬──────┘                         │
│         │                   │                                │
│  ═══════╪═══════════════════╪══════════════════════════════  │
│         │                   │                                │
│  ┌──────▼───────────────────▼──────┐                         │
│  │           cAdvisor :8081        │   Metric Exposure       │
│  │   (discovers all containers)    │                         │
│  └──────────────┬──────────────────┘                         │
│                 │  scrape /metrics                            │
│  ┌──────────────▼──────────────────┐                         │
│  │       Prometheus :9090          │   Metric Storage        │
│  │   (time-series database)        │                         │
│  └──────────────┬──────────────────┘                         │
│                 │                                            │
│  ┌──────────────▼──────────────────┐                         │
│  │         Grafana :3000           │   Visualization         │
│  │   (dashboards + log explorer)   │                         │
│  └──────────────▲──────────────────┘                         │
│                 │                                            │
│  ┌──────────────┴──────────────────┐                         │
│  │          Loki :3100             │   Log Storage           │
│  └──────────────▲──────────────────┘                         │
│                 │                                            │
│  ┌──────────────┴──────────────────┐                         │
│  │       Promtail (agent)          │   Log Collection        │
│  │   (reads Docker log files)      │                         │
│  └─────────────────────────────────┘                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> A detailed architecture walkthrough is available in [`architecture/stack_architecture.md`](architecture/stack_architecture.md).

---

## Phases

| Phase | Title | Status |
| --- | --- | --- |
| [Architecture](architecture/stack_architecture.md) | Design stack architecture & component selection | 🔄 In progress |
| [Phase 1](phases/phase-1-environment-setup.md) | Environment preparation & Docker configuration | 🔄 In progress |
| [Phase 2](phases/phase-2-service-configuration.md) | Service configuration (Prometheus, Promtail, Loki) | 🔜 Planned |
| [Phase 3](phases/phase-3-docker-compose.md) | Docker Compose orchestration & deployment | 🔜 Planned |
| [Phase 4](phases/phase-4-grafana-setup.md) | Grafana data sources & dashboard configuration | 🔜 Planned |
| [Phase 5](phases/phase-5-analysis.md) | Performance analysis & component diagnostics | 🔜 Planned |
| [Phase 6](phases/phase-6-alerting.md) | Alert rules & notification channels | 🔜 Planned |
| [Phase 7](phases/phase-7-hardening.md) | Stack hardening & production best practices | 🔜 Planned |

> **Note:** Phases 1–5 document the original lab implementation. Phases 6–7 are extensions that elevate the project beyond the academic scope into production-readiness.

---

## Key Objectives

**Single-command deployment** — The entire 7-container stack (application + monitoring) is deployed via a single `docker compose up -d` command. All services are pre-configured through mounted configuration files, requiring zero manual post-boot intervention.

**Three pillars of observability** — The stack implements the three pillars of modern observability: metrics (Prometheus + cAdvisor), logs (Loki + Promtail), and visualization (Grafana), unified under a single-pane-of-glass dashboard.

**Container-native metric collection** — cAdvisor automatically discovers every running container and exposes CPU, memory, filesystem, and network metrics in Prometheus-compatible format, requiring no application-level instrumentation.

**Centralized log aggregation** — Promtail harvests Docker container logs directly from the host filesystem (`/var/lib/docker/containers`), labels them by container name, and ships them to Loki for indexed storage and ad-hoc querying through Grafana's Explore interface.

**Real-time performance dashboards** — A pre-built Grafana dashboard (ID 193) provides immediate visibility into running container count, total CPU usage, memory consumption, and per-container network I/O — all refreshing at 10-second intervals.

**Incident diagnosis workflow** — The documentation includes a structured 3-step diagnostic methodology: (1) identify anomalies in Prometheus metrics, (2) correlate with Loki logs in the same time window, (3) determine root cause by cross-referencing both data sources.

---

## Stack Components

| Service | Image | Port | Role |
| --- | --- | --- | --- |
| WordPress | `wordpress` | 8080 | Web application (monitored target) |
| MySQL | `mysql:5.7` | 3306 | Database backend |
| Prometheus | `prom/prometheus` | 9090 | Time-series metric database |
| Grafana | `grafana/grafana` | 3000 | Visualization & dashboards |
| Loki | `grafana/loki` | 3100 | Log aggregation engine |
| Promtail | `grafana/promtail` | — | Log collection agent |
| cAdvisor | `gcr.io/cadvisor/cadvisor` | 8081 | Container metric exporter |

---

## Notable Technical Challenges Solved

**Docker daemon crash from missing Loki plugin** — An initial attempt to configure the Loki logging driver directly in `/etc/docker/daemon.json` prevented the Docker daemon from starting (`plugin "loki" not found`). The solution was to delegate log collection to Promtail instead of the daemon, restoring `daemon.json` to `{}` and using file-based log harvesting.

**Docker Compose V1 vs V2 incompatibility** — The legacy `docker-compose` (Python-based V1) failed with `ModuleNotFoundError: No module named 'distutils'` on Python 3.12+. Migration to Docker Compose V2 (`docker-compose-v2` package) resolved the issue and all commands were updated to the `docker compose` (space) syntax.

**Permission denied on Docker socket** — Non-root execution of `docker compose` failed with `PermissionError(13, 'Permission denied')` on `/var/run/docker.sock`. Resolved by prepending `sudo` to all Docker commands (documented as a conscious decision vs. adding the user to the `docker` group).

**IPv6 network failures during image pull** — Docker defaulted to IPv6 connections on the VM, which lacked valid IPv6 connectivity, causing intermittent `connect: network is unreachable` errors. Disabling IPv6 at the OS level via `/etc/sysctl.conf` (`net.ipv6.conf.all.disable_ipv6=1`) forced all traffic through IPv4, permanently resolving download failures.

---

## Repository Structure

```
docker-monitoring-stack/
│
├── LICENSE
├── README.md
├── SECURITY.md
│
├── .github/
│   ├── CODE_OF_CONDUCT.md
│   ├── CONTRIBUTING.md
│   └── ISSUE_TEMPLATE/
│       ├── config.yml
│       ├── bug-report.yml
│       └── phase-suggestion.yml
│
├── architecture/
│   └── stack_architecture.md
│
├── configs/
│   ├── docker-compose.yml
│   ├── prometheus/
│   │   └── prometheus.yml
│   ├── promtail/
│   │   └── config.yml
│   └── grafana/
│       └── dashboards/
│           └── docker-monitoring.json
│
├── phases/
│   ├── phase-1-environment-setup.md
│   ├── phase-2-service-configuration.md
│   ├── phase-3-docker-compose.md
│   ├── phase-4-grafana-setup.md
│   ├── phase-5-analysis.md
│   ├── phase-6-alerting.md
│   └── phase-7-hardening.md
│
└── docs/
    ├── troubleshooting.md
    └── diagnostic-methodology.md
```

---

## Roadmap

- Phase 6: Configure Prometheus alert rules (`alert.rules.yml`) with notification channels (Slack/email webhooks)
- Phase 7: Stack hardening — resource limits, Grafana auth, HTTPS reverse proxy, `.env` parameterization
- Future: Integrate Tempo for distributed tracing (completing the third observability pillar)
- Future: Add Node Exporter for host-level metrics beyond container scope

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/alejandroZ345/docker-monitoring-stack.git
cd docker-monitoring-stack/configs

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

## About

**Alejandro Zavala** — Systems Engineer & Cybersecurity Professional
ISC2 Certified in Cybersecurity (CC) · Specialization: SOC Operations, Infrastructure Monitoring, Technical Documentation
[linkedin.com/in/alejandro-zavala-zenteno](https://www.linkedin.com/in/alejandro-zavala-zenteno) · [ISC2 Badge](https://www.credly.com/badges/fe6eeefe-d684-45fa-8f57-9f7d025db2d6/public_url)
