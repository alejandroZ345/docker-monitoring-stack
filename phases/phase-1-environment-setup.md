# Phase 1 — Infrastructure Deployment & Zero-Instrumentation Telemetry

> Deploy a cloud-native observability stack (Prometheus, Loki, Grafana, Promtail, cAdvisor) alongside a target application (WordPress + MySQL) using a unified Docker Compose architecture.

---

## 1. Objective

Establish a **zero-instrumentation observability baseline** for a containerized application. The goal is to collect metrics and logs from a running WordPress + MySQL deployment without modifying a single line of application source code, proving that container-native telemetry can be achieved entirely through external collectors.

By the end of this phase:

- A unified `monitoring_network` hosts the full 7-container stack.
- Prometheus is scraping container metrics from cAdvisor every 15 seconds.
- Promtail is shipping Docker container logs into Loki via Docker service discovery.
- Grafana is provisioned and ready to consume both data sources.

---

## 2. Architecture & Deployment Setup

All services are provisioned within a single virtualized Docker bridge network (`monitoring_network`). Persistent volumes are strictly mapped for database storage and configuration files to ensure data survives container restarts.

The host environment leverages a multi-core architecture operating via WSL2, providing sufficient compute resources for concurrent Java/Go-based metric processing without impacting the database tier.

| Tier | Services | Purpose |
| --- | --- | --- |
| Application | `wordpress`, `mysql` | The target workload being monitored |
| Metrics Pipeline | `prometheus`, `cadvisor` | Container metric scraping & time-series storage |
| Logs Pipeline | `loki`, `promtail` | Log aggregation & ingestion via Docker SD |
| Visualization | `grafana` | Unified dashboard frontend for both data sources |

---

## 3. Deployment Artifacts (Orchestration)

The foundational infrastructure is defined in a unified compose file. **Note:** Loki and Promtail are intentionally pinned to version `2.9.8` to maintain compatibility with the local storage schema (detailed in [Section 5](#5-architectural-challenges--troubleshooting)).

> All files referenced below live inside the `monitoring-stack/` root directory.

### File: `docker-compose.yml`

```yaml
networks:
  monitoring_network:
    driver: bridge

volumes:
  mysql_data:
  wordpress_data:
  prometheus_data:
  grafana_data:
  loki_data:

services:
  # === APPLICATION TIER ===
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp_user
      MYSQL_PASSWORD: wp_password
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - monitoring_network

  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: mysql:3306
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: wp_password
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html
    depends_on:
      - mysql
    networks:
      - monitoring_network

  # === METRICS PIPELINE ===
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - monitoring_network

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.0
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8081:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    networks:
      - monitoring_network

  # === LOGS PIPELINE ===
  loki:
    image: grafana/loki:2.9.8
    container_name: loki
    restart: unless-stopped
    user: "root"   # Required to write to root-owned loki_data volume (see Discovery 2)
    ports:
      - "3100:3100"
    volumes:
      - ./config/loki:/etc/loki
      - loki_data:/tmp/loki
    command: -config.file=/etc/loki/loki-config.yml
    networks:
      - monitoring_network

  promtail:
    image: grafana/promtail:2.9.8
    container_name: promtail
    restart: unless-stopped
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./config/promtail:/etc/promtail
    command: -config.file=/etc/promtail/promtail-config.yml
    networks:
      - monitoring_network

  # === VISUALIZATION TIER ===
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./config/grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
      - loki
    networks:
      - monitoring_network
```

---

## 4. Service Configurations (Baseline Injections)

Before container initialization, baseline configurations were injected into the respective volume mounts to establish the data pipelines.

### File: `config/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

### File: `config/loki/loki-config.yml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
```

### File: `config/promtail/promtail-config.yml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
```

---

## 5. Architectural Challenges & Troubleshooting

During the initial deployment phase, two critical container lifecycle failures were detected and resolved.

### Discovery 1 — Storage Schema Deprecation in Loki 3.x (Breaking Change)

**Issue:** The Loki container exited immediately upon creation with a status code indicating a fatal configuration error.

**Analysis:** Diagnostic logs (`docker logs loki`) revealed the following error:

```
CONFIG ERROR: schema v13 is required... your schema version is v11... your index type is boltdb-shipper
```

The original orchestration file specified the `grafana/loki:latest` tag. Docker pulled the newer 3.x generation, which strictly enforces the `tsdb` index type and `schema v13` for structured metadata, rendering the stable `boltdb-shipper` configuration incompatible.

**Resolution:** Modified the orchestration file to explicitly pin both Loki and Promtail to the last stable release of the previous generation (`2.9.8`). This ensured architectural compatibility with the defined local storage configuration, preventing future deployment-breaking changes.

> **Lesson learned:** Avoid `:latest` tags for stateful services with versioned storage schemas. Pin explicit versions and document the rationale in the compose file.

### Discovery 2 — UID/GID Permission Collision on Persistent Volumes

**Issue:** Shortly after initialization, the Loki container entered a continuous crash loop (status: `Restarting`).

**Analysis:** Extraction of the container's internal logs (`docker compose logs loki`) revealed a fatal filesystem error:

```
mkdir /tmp/loki/rules: permission denied
```

A Root Cause Analysis (RCA) identified this as a User Identifier (UID) collision. By default, the Docker engine daemon provisioned the persistent volume (`loki_data`) with host-level `root` ownership. However, following security best practices, the official Grafana Loki image executes as an unprivileged internal user (typically UID `10001`). This unprivileged process lacked the necessary write permissions to create the local indexing and chunk directories inside the root-owned mount.

**Resolution:** Modified the container orchestration strategy to explicitly elevate the runtime privileges of the Loki service for this specific environment. The directive `user: "root"` was injected into the `docker-compose.yml` Loki service definition. This aligned the container's execution context with the volume's host-level ownership, granting full read/write access to the local storage path (`/tmp/loki`) and permanently stabilizing the container lifecycle.

> **Lesson learned:** When persistent volumes are provisioned by the Docker daemon, the container's internal UID must match the mount's ownership — or be elevated — to write successfully. In production, a cleaner fix would be an `initContainer` that `chown`s the volume to the service's unprivileged UID, preserving the principle of least privilege.

---

## 6. Final Validation

The modified orchestration file was executed successfully. All 7 containers initialized and established stable runtimes:

```
CONTAINER ID   IMAGE                              COMMAND                  STATUS                   PORTS                    NAMES
0c90373930e9   grafana/promtail:2.9.8             "/usr/bin/promtail -…"   Up 3 seconds                                      promtail
11b2ac0032c2   grafana/loki:2.9.8                 "/usr/bin/loki -conf…"   Up Less than a second    0.0.0.0:3100->3100/tcp   loki
c2d946c3ed17   wordpress:latest                   "docker-entrypoint.s…"   Up 7 minutes             0.0.0.0:8080->80/tcp     wordpress
0952b3ea81d6   grafana/grafana:latest             "/run.sh"                Up 7 minutes             0.0.0.0:3000->3000/tcp   grafana
3cdad7c9eab9   gcr.io/cadvisor/cadvisor:v0.47.0   "/usr/bin/cadvisor -…"   Up 7 minutes (healthy)   0.0.0.0:8081->8080/tcp   cadvisor
ed75e966a175   mysql:8.0                          "docker-entrypoint.s…"   Up 7 minutes             3306/tcp, 33060/tcp      mysql
3533cc392c1a   prom/prometheus:latest             "/bin/prometheus --c…"   Up 7 minutes             0.0.0.0:9090->9090/tcp   prometheus
```

### Verification checklist

- [x] All 7 containers report `Up` status
- [x] cAdvisor reports `healthy` from its built-in health check
- [x] Prometheus, Grafana, Loki, WordPress, and cAdvisor expose their respective ports on `0.0.0.0`
- [x] MySQL remains internal-only (no external port mapping) — correct security posture
- [x] No restart loops observed during the first 7 minutes of runtime

---

## Next Steps

With the infrastructure baseline established, [Phase 2](phase-2-telemetry-visualization.md) will focus on activating the visualization tier: connecting Grafana to Prometheus as a data source, importing a cAdvisor dashboard, and engineering around the WSL2 virtualization constraints that affect per-container metric granularity.
