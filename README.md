# Homelab Infrastructure

**[Portfolio & Case Studies](https://singletons.craft.me/resume_webpage)** · **[LinkedIn](https://www.linkedin.com/in/msingleton18)**

Production-grade self-hosted infrastructure running real services for my household — with the same discipline I'd apply to a production SRE environment: defined SLOs, documented incident response, zero-trust secrets, and a monitoring stack built to catch problems before people do. Live and actively maintained since December 2024.

## Highlights

- **MTTR:** 30 min → under 5 min
- **Operational toil:** 10+ hrs/week → under 2 hrs/week (automated SSL renewal, updates, backups, health checks)
- **Availability:** 100% during DNS maintenance windows (Pi-hole failover)
- **Observability:** Prometheus + 8 exporters across 3 hosts, Loki log aggregation, Grafana unified dashboards

## Recent Writing

- [Standing Up My Personal Media Server - Debugging Docker Desktop's macOS Mount Layer](https://www.linkedin.com/pulse/standing-up-my-personal-media-server-debugging-docker-singleton-oleac) — an eight-step root-cause chain through Docker Desktop's virtualized mount layer, and the native reinstall that fixed it for good.
- [Lessons from my CKA prep and Exam](https://www.linkedin.com/pulse/lessons-from-my-cka-prep-exam-marcus-singleton-k89bc) — what a two-point exam miss exposed about untested Kubernetes gaps, and closing them for a 90% retake.

## Architecture

**Reverse Proxy & TLS** — Traefik with automated Let's Encrypt certificate management (DNS-01 challenge), intelligent routing across distributed services.

**Secrets** — Hierarchical 1Password CLI integration. No hardcoded credentials anywhere in the stack.

**DNS** — Pi-hole with automatic failover.

**Observability** — Prometheus scraping 8 exporters (Node, cAdvisor, UniFi Poller, Pi-hole, Traefik, Proxmox) at 30s intervals; Loki + Promtail for log aggregation (14-day retention); Grafana dashboards combining both data sources.

**Kubernetes** — 3-node K3s cluster on Proxmox, live and reboot-tested. Flux CD GitOps *(in progress)*.

**Backup** — Restic, shipping offsite to S3.

**Cloud** — AWS backup/cloud-integration workflow *(in progress)*.

## Repository Structure

```
homelab-infrastructure/
├── aws-infrastructure/      # AWS backups & cloud integration workflow (in progress)
├── kubernetes-cluster/      # K3s cluster (live) + Flux CD GitOps (in progress)
├── monitoring-stack/        # Prometheus + Grafana
└── traefik-reverse-proxy/   # Traefik config + Let's Encrypt automation (documented — see its README)
```

## Tech Stack

Docker · Proxmox · Traefik · Prometheus · Grafana · Loki · Promtail · Pi-hole · 1Password CLI · K3s · Flux CD · AWS · Restic · UniFi · Linux (Ubuntu) · Bash

## Status

`traefik-reverse-proxy` is documented and production-ready. `monitoring-stack` is live. `kubernetes-cluster`'s K3s cluster is live and reboot-tested; its Flux CD GitOps layer is still in progress. `aws-infrastructure` remains active work in progress — scaffolding is in place, implementation is ongoing.


*Built to learn Site Reliability Engineering by doing it — every design decision follows a framework around Security, Observability, Resiliency, Automation, Portability, and Reachability.*
