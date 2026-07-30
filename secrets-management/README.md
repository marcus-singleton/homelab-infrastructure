# Zero-Trust Secrets Management with 1Password

<div align="center">

![1Password](https://img.shields.io/badge/1Password-Secrets-blue?logo=1password)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)
![Status](https://img.shields.io/badge/Status-Production-success)

*No credential ever touches a config file — every secret is a runtime pointer, not a hardcoded value*

[Features](#-features) • [Quick Start](#-quick-start) • [Architecture](#-architecture)

</div>


## 🎯 Project Overview

The secrets pattern used across every credential in this stack:
- ✅ Zero hardcoded credentials anywhere in the repo
- ✅ 1Password CLI + Service Accounts for automation
- ✅ Git-safe config — references only, never real secret values
- ✅ One pattern, reused identically for every credential added since

### The Problem

Common self-hosted setups present:
- ❌ API tokens hardcoded directly in `docker-compose.yml`
- ❌ Secrets committed to git history — hard to fully scrub once pushed
- ❌ No audit trail on who or what accessed a credential, or when
- ❌ A different ad-hoc secrets approach per service

### The Solution

A single pattern applied everywhere a credential is needed:
- **1Password Service Accounts** — scoped, revocable, audit-logged access, not a personal vault login shared across automation
- **Config files reference secrets by pointer only** (`op://vault/item/field`) — safe to commit to git, because the reference contains no actual secret material
- **Real secret resolved only at runtime**, injected as an environment variable, never written to disk in plaintext


## ✨ Features

### Security
- 🔒 **Zero hardcoded credentials** — verified across every service in the stack
- 🔑 **Scoped, revocable access** via Service Accounts, not shared personal logins
- 🛡️ **Git-safe references** — `op://` pointers carry no secret material

### Portability
- 🔄 **Vendor-agnostic principle** — the same pattern works with HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault; 1Password is the implementation, not the architecture
- 🎯 **Reused identically** across every credential added since — the Cloudflare DNS API token today, AWS IAM credentials next (once Restic → S3 backups go live)


## 🏗️ Architecture

```
[Config file — op:// pointer]
    → safe to commit to git
         ↓ at service start
[1Password CLI: op read] → [Service Account]
         ↓
[Real secret resolved, injected as env var — never written to disk]
         ↓
[Container CREATED with secret baked in]
```

**Key Design Decisions:**
- **Pointer-based references** — the only thing that ever touches a config file or git history is a pointer to where the secret lives, never the secret itself
- **Runtime resolution only** — nothing sensitive exists on disk in plaintext, at rest or in transit through the filesystem
- **Recreate, not restart** — see the lesson below; this distinction is load-bearing for the whole pattern to actually work


## 🛠️ Tech Stack

| Component | Purpose |
|-----------|---------|
| 1Password CLI (`op`) | Runtime secret resolution |
| 1Password Service Accounts | Scoped, revocable automation access |
| Docker Compose | Environment variable injection at container creation |


## 🚀 Quick Start

This isn't a standalone service to deploy — it's the pattern every other service in this repo uses to handle its credentials. To apply it to a new service:

**1. Store the secret in 1Password**, under a dedicated vault/item for the service.

**2. Reference it by pointer in the config** — safe to commit:
```bash
CF_DNS_API_TOKEN=op://Traefik/Cloudflare DNS API Token/password
```

**3. Resolve and inject at container creation:**
```bash
CF_DNS_API_TOKEN=$(op read "op://Traefik/Cloudflare DNS API Token/password") \
  docker compose up -d
```


## 🐛 Real Lessons Learned

### Docker Doesn't Hot-Reload Environment Variables

The first time this pattern hit production, fixing a broken token injection and running `docker compose restart` silently didn't pick up the new value — the service kept logging that the variable was unset. The cause: Docker bakes environment variables into a container at **creation**, not at **start**. `restart` restarts the same container with its original environment; only `docker compose down` followed by a fresh `up` (with the variable injected again) actually recreates it with the new value.

A small, easy-to-miss distinction — and exactly the kind of thing that looks obvious in hindsight but costs real debugging time the first time a secret needs rotating in production.

### Verifying AI-Assisted Troubleshooting Against Primary Sources

Debugging a stalled `docker compose up` in this stack, an AI coding assistant confidently claimed that 1Password's `op run` doesn't resolve `op://` secret references from `.env` files, and proposed restructuring the config around that claim. Before implementing the change, I checked it against 1Password's own CLI documentation — the claim was wrong. `op run --env-file=.env -- <command>` does resolve `.env` references; the actual issue was a missing `--env-file` flag, not a fundamental limitation. Pushing back and verifying before acting caught what would have been an unnecessary architecture change.

As AI-assisted troubleshooting becomes a standard part of the workflow, the differentiator isn't using the tools — it's verifying their output against primary sources before anything ships on top of it.


## 🎓 With This Project, I Practiced

**Technical Skills:**
- Secrets lifecycle management (storage, reference, runtime resolution, rotation)
- Docker container lifecycle — creation vs. restart semantics
- CLI-based API integration (1Password)

**DevOps Practices:**
- Zero-trust secrets handling as a repeatable pattern, not a one-off fix
- Treating a debugging surprise as a documented lesson, not just a fixed bug


## 💼 Professional Context

**Built as part of my SRE learning journey.**

This is the concrete artifact behind security as a positioning pillar — not a claim, a pattern implemented consistently across every credential in the stack, with a documented failure mode and fix.

See my [complete portfolio](https://github.com/marcus-singleton) for more projects.


<div align="center">

**🔗 [View Project](https://github.com/marcus-singleton/homelab-infrastructure) • [Connect](https://linkedin.com/in/msingleton18)**

</div>
