# Distributed Observability Stack: Prometheus, Grafana & Loki

<div align="center">

![Prometheus](https://img.shields.io/badge/Prometheus-Metrics-E6522C?logo=prometheus)
![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800?logo=grafana)
![Loki](https://img.shields.io/badge/Loki-Log_Aggregation-blue?logo=grafana)
![1Password](https://img.shields.io/badge/1Password-Secrets-blue?logo=1password)
![Status](https://img.shields.io/badge/Status-Live-success)

*Monitoring infrastructure that survives the failures it's designed to detect*

[Features](#-features) • [Architecture](#-architecture) • [SLO Dashboard](#-slo-tracking) • [Lessons Learned](#-real-lessons-learned)

</div>

## 🎯 Project Overview

A distributed observability stack spanning three independently powered hosts, tracking real SLOs with error-budget math instead of just "is it up":

- ✅ **8 Prometheus targets, 100% UP** — hosts, containers, DNS, network, and the hypervisor
- ✅ **4.4M+ log lines** aggregated via Loki, queryable with LogQL across all three hosts
- ✅ **6+ Grafana dashboards**, including a hand-written SLO tracker with error-budget burn-down
- ✅ **Zero hardcoded credentials** — 1Password three-tier service-account chain end to end
- ✅ **Full network visibility** via UniFi Poller (clients, APs, gateway throughput)

### The Problem

Before this stack existed:
- ❌ No way to answer "what's my Pi-hole uptime this month?" — no metrics collection at all
- ❌ Logs scattered across three hosts, manual SSH + `grep` to troubleshoot anything
- ❌ No SLO tracking, no error budget, no data-driven read on reliability — just vibes
- ❌ The common homelab failure mode: monitoring that lives on the same host it's watching, so a bad night on that host produces silence instead of an alert

### The Solution

Observability infrastructure has to stay independent of what it observes — not as an aspiration, structurally. So the stack is split by function across three hosts, with the hypervisor watched but never doing the watching:

- Prometheus scraping 8 targets across all three hosts
- Loki + Promtail centralizing logs so a single host going down doesn't also take out the ability to diagnose it
- A hand-built SLO dashboard (PromQL, not a pre-built import) with real error-budget math
- Zero-trust secrets via a 1Password three-tier service-account chain — nothing lands in a config file or git history

## 🏗️ Architecture

```
┌─────────────────────┐    ┌─────────────────────┐    ┌────────────────────┐
│  Pi1 — Log Layer     │    │  Pi2 — Metrics/Edge │    │  Proxmox Host      │
│  <PI1_IP>            │    │  <PI2_IP>            │    │  <PVE_IP>          │
├─────────────────────┤    ├─────────────────────┤    ├────────────────────┤
│  - Loki         ────────▶ │  - Prometheus        │◀──── - Node Exporter │
│  - Promtail          │    │  - Grafana           │    │                    │
│  - Pi-hole           │    │  - Traefik           │    │  Instrumented,     │
│  - Pi-hole Exporter  │    │  - UniFi Poller      │    │  not hosting — it  │
│  - Node Exporter ───────▶ │  - PVE Exporter      │    │  gets watched like │
│                       │    │  - cAdvisor          │    │  everything else,  │
│                       │    │  - Node Exporter     │    │  it doesn't get to │
│                       │    │                       │    │  do the watching.  │
└─────────────────────┘    └──────────┬──────────┘    └────────────────────┘
                                       │
                                       ▼
                          ┌─────────────────────────┐
                          │   UniFi Cloud Gateway    │
                          │   (Network Metrics)      │
                          └─────────────────────────┘
```

**Why this split, specifically:**
- **Log layer isolated from metrics/edge** — if Pi2 goes down during a Traefik upgrade, Pi1 keeps collecting logs. This isn't theoretical: it happened during the build and confirmed the architecture decision on the spot.
- **Hypervisor instrumented, not hosting** — if Prometheus and Grafana lived on the same Proxmox box they're supposed to be watching, a bad night on that host produces silence, not an alert, at exactly the moment it matters most.
- **Redundancy by construction, not configuration** — metrics and logs failing independently was a design goal, not a side effect.

## ✨ Features

### Observability
- 📊 **8 Prometheus targets** across hosts, containers, DNS, and network — 100% UP
- 📝 **4.4M+ log lines** in Loki, queryable via LogQL, correlated across all three hosts
- 📈 **6+ Grafana dashboards** — host/container metrics, DNS stats, network visibility, and a custom SLO tracker
- 🌐 **UniFi Poller integration** — client tracking, AP stats, gateway throughput, all in Prometheus

### Reliability Engineering
- 🎯 **Real SLOs with error-budget math**, not vanity uptime numbers on a dashboard nobody reads
- 🔢 **Hand-written PromQL**, not a copy-pasted community dashboard — see [SLO Tracking](#-slo-tracking) below

### Security
- 🔒 **Zero hardcoded credentials** anywhere in the stack
- 🔑 **1Password three-tier service-account chain** — a Pi2 service account can't directly read monitoring credentials; it has to go through the chain
- 🛡️ **Runtime-only secret resolution** via `op run` — secrets never touch disk, never appear in git history

## 🎯 SLO Tracking

Service Level Objectives, each with a real measurement window:

| Service | SLI (Indicator) | SLO (Objective) | Window |
|---|---|---|---|
| Pi-hole DNS | Query success rate | 99.9% | 30 days |
| Traefik | Request success rate (2xx) | 99.5% | 7 days |
| All hosts | Uptime (`up == 1`) | 99.9% | 30 days |
| Prometheus | Scrape success rate | 99.5% | 7 days |

The dashboard's four panels, in the PromQL that actually drives them:

**Service Availability** — what fraction of the last 7 days was this reachable:
```promql
avg_over_time(up{job="node-exporter"}[7d]) * 100
```

**Error Budget Remaining** — how much of a 99.9% SLO's ~43-minutes-per-month downtime budget is left:
```promql
((1 - 0.999) * 30 * 24 * 60 - downtime_minutes_this_month) /
 ((1 - 0.999) * 30 * 24 * 60) * 100
```

**Traefik Request Success Rate**:
```promql
sum(rate(traefik_service_requests_total{code=~"2.."}[5m])) /
sum(rate(traefik_service_requests_total[5m])) * 100
```

These are built from scratch rather than imported, deliberately — understanding the underlying query matters more here than a working dashboard alone would.

## 🛠️ Tech Stack

| Component | Purpose |
|---|---|
| Prometheus | Time-series metrics collection (8 targets) |
| Grafana | Dashboards, visualization, SLO tracking |
| Loki + Promtail | Centralized log aggregation and shipping |
| Traefik | Reverse proxy, automated Let's Encrypt TLS |
| UniFi Poller | Network infrastructure metrics |
| Node Exporter | Host metrics — deployed on all three hosts |
| cAdvisor | Container resource metrics |
| PVE Exporter | Proxmox VM/container/storage metrics |
| Pi-hole Exporter | DNS query and blocklist metrics |
| 1Password CLI | Runtime secrets resolution, zero-trust pattern |

## 🔐 Secrets: The Nested 1Password Chain

Every credential in this stack resolves through a two-step chain instead of a single flat vault lookup:

```bash
# Step 1: host service account → Automation vault → monitoring SA token
export OP_SERVICE_ACCOUNT_TOKEN=$(cat ~/.config/op/pi2-sa-token)
MONITORING_SA_TOKEN=$(op read "op://Automation/monitoring-service-account/credential")

# Step 2: monitoring SA → Monitoring vault → the actual credential
export OP_SERVICE_ACCOUNT_TOKEN="$MONITORING_SA_TOKEN"
export UP_UNIFI_DEFAULT_USER=$(op read "op://Monitoring/unifi-controller/username")
```

The host's own service account can't read monitoring credentials directly — it has to go through the monitoring service account first, which itself only has read-only access to the vault it needs. Scope stays narrow at every hop.

## 🐛 Real Lessons Learned

### Docker's `localhost` Isn't What You Think Inside a Container

Prometheus couldn't scrape the PVE Exporter running in Docker on the same host it was on. `localhost:9221` from inside the Prometheus container refers to the container, not the host — the fix was pointing the scrape target at the Docker service name (`pve-exporter:9221`) instead. A small distinction that costs real debugging time the first time it bites.

### Traefik v3's Breaking Change on the Metrics Endpoint

The Traefik metrics scrape returned a flat 404 after upgrading to v3, which requires an explicit `entryPoint` binding for the metrics endpoint that earlier versions didn't. Reading the migration guide up front would have saved the hour spent guessing at "wrong endpoint vs. metrics not enabled."

### Community Dashboards Are a Starting Point, Not Plug-and-Play

Imported UniFi dashboards rendered every panel as "No data" despite Prometheus confirming the UniFi Poller target was UP and scraping. Root cause: UniFi Poller v2 renamed its metric prefix (`unpoller_*` → `unifipoller_*`), and the dashboard JSON still referenced the old one. Fixed with a global find-and-replace — but the real lesson is to check actual metric names in Prometheus before importing a dashboard, not after debugging why it's empty.

### Verifying AI-Assisted Troubleshooting Against Primary Sources

While building the secrets chain, an AI troubleshooting assistant confidently claimed `op run` couldn't resolve secrets from a `.env` file — plausible-sounding, and wrong. Checking it against 1Password's own CLI documentation before restructuring anything around that claim showed the actual issue was a missing `--env-file` flag. Treating AI output as a hypothesis to verify rather than an instruction to follow caught the error before it shipped.

## 🎓 With This Project, I Practiced

**Technical Skills:**
- PromQL query development, from basic availability ratios to error-budget math
- LogQL and cross-host log correlation
- Distributed systems design under a real resiliency constraint (observability must outlive what it observes)
- Docker networking — container-name resolution vs. `localhost`
- Zero-trust secrets architecture with scoped, chained service accounts

**Problem Solving:**
- Hypothesis-driven debugging across container networking, API auth, and third-party metric-naming changes
- Verifying AI-generated troubleshooting claims against primary-source documentation before acting on them

## 🚀 What's Next

- **Alertmanager** — dashboards currently tell the truth if you go looking; they can't yet tell it unprompted. That's the gap between a dashboard and an on-call system, and it's the next thing closing.
- Loki alerting rules on top of the existing log aggregation
- K3s cluster metrics via Prometheus Operator / ServiceMonitor CRDs, once the [kubernetes-cluster](../kubernetes-cluster) GitOps layer lands

## 💼 Professional Context

**Built as part of my SRE learning journey.**

This is the flagship observability artifact in this repo — real SLOs with error-budget tracking, a distributed architecture that survives the failure it's watching for, and zero-trust secrets handling applied consistently rather than bolted on at the end.

See my [complete portfolio](https://github.com/marcus-singleton) for more projects.

<div align="center">

**🔗 [View Project](https://github.com/marcus-singleton/homelab-infrastructure) • [Connect](https://linkedin.com/in/msingleton18)**

</div>
