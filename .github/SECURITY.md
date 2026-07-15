# Security Policy

## Purpose

This repository is an **educational home lab project** for learning Docker-based observability. It is **not intended for production deployment** without the hardening changes described below.

## Reporting a Vulnerability

If you discover a security issue in the configurations or documentation:

1. **Do not** open a public issue.
2. Contact the maintainer directly via [LinkedIn](https://www.linkedin.com/in/alejandro-zavala-zenteno).
3. Include a description of the issue and, if possible, steps to reproduce.

## Known Security Trade-offs (Lab Context)

These are intentional, documented decisions made for a local lab environment — not oversights:

| Trade-off | Where it's introduced | Why it's acceptable here |
| --- | --- | --- |
| Default Grafana credentials (`admin/admin`) | `docker-compose.yml` (Grafana env vars) | No external exposure; localhost-only lab. |
| MySQL credentials hardcoded in plaintext | `docker-compose.yml` (MySQL env vars) | Same as above; not committed with real secrets. |
| Loki container runs as `user: "root"` | `docker-compose.yml` (Loki service) | Resolves a UID/GID collision on the `loki_data` volume (see [Phase 1, Discovery 2](../phases/phase-1-environment-setup.md#discovery-2--uidgid-permission-collision-on-persistent-volumes)). In production, an `initContainer` should `chown` the volume instead. |
| cAdvisor runs `privileged: true` with `/var/run/docker.sock` mounted | `docker-compose.yml` (cAdvisor service) | Required for container discovery and metric collection; grants broad host visibility. |
| No inter-service TLS | All services | Traffic stays inside the isolated `monitoring_network` Docker bridge. |

## Planned Hardening

These trade-offs are tracked as a roadmap item — see [Stack Hardening](../README.md#roadmap) in the README — covering resource limits, non-default Grafana auth, an HTTPS reverse proxy, and `.env`-based secret parameterization.

## Supported Versions

This is a lab/portfolio project without formal release versioning. Security fixes, if any, are applied to the `main` branch only.
