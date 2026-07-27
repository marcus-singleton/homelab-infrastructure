# Traefik Reverse Proxy with Let's Encrypt & 1Password

<div align="center">

![Traefik](https://img.shields.io/badge/Traefik-v3.2-blue?logo=traefikproxy)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)
![Let's Encrypt](https://img.shields.io/badge/Let's_Encrypt-Automated-green)
![1Password](https://img.shields.io/badge/1Password-Secrets-blue?logo=1password)
![Status](https://img.shields.io/badge/Status-Production-success)

*Enterprise-grade reverse proxy for home lab infrastructure*

[Features](#-features) • [Quick Start](#-quick-start) • [Architecture](#️-architecture)

</div>

---

## 🎯 Project Overview

This project implements a production-ready reverse proxy using Traefik that provides:
- ✅ Automated HTTPS certificates (Let's Encrypt)
- ✅ Zero-trust secrets management (1Password)
- ✅ File-based routing configuration
- ✅ Automatic certificate renewal
- ✅ No browser security warnings

### The Problem

Self-hosted services typically present:
- ❌ Browser certificate warnings
- ❌ Multiple IP addresses and ports to remember
- ❌ No centralized access control
- ❌ Manual certificate management

### The Solution

A centralized reverse proxy that:
- Acts as single HTTPS entry point
- Routes traffic based on domain names
- Automatically manages SSL certificates
- Securely stores all credentials
- Provides zero-downtime updates

### Real-World Application

This architecture mirrors production environments:
- Used by startups for staging/dev environments
- Implements edge router patterns
- Demonstrates microservices access patterns
- Shows zero-trust security principles

---

## ✨ Features

### Security
- 🔒 **Automated HTTPS** - Let's Encrypt with DNS-01 challenge
- 🔑 **Secrets Management** - 1Password Service Account integration
- 🛡️ **Zero Hardcoded Credentials** - All secrets externalized
- 🔐 **TLS 1.2+ Only** - Modern encryption standards

### Automation
- ⚡ **Auto Certificate Renewal** - 30 days before expiration
- 🔄 **Dynamic Configuration** - Changes without restarts
- 🎯 **Zero-Downtime Updates** - Rolling updates

### Operations
- 📈 **Built-in Dashboard** - Real-time monitoring
- 📝 **Comprehensive Logging** - Debug and access logs
- 🎛️ **Middleware Support** - Auth, rate limiting, headers
- 🔍 **Health Checks** - Automatic backend verification

---

## 🏗️ Architecture

```markdown
[Internet/Local Network]
         ↓
    [Cloudflare DNS]
         ↓
    [Pi-hole (Local DNS)]
         ↓
┌─────────────────────────────────┐
│   Traefik Edge Node (RPi5)     │
│   • TLS Termination            │
│   • Host-based Routing         │
│   • 1Password Integration      │
└─────────────────────────────────┘
         ↓
    [Backend Services]
    - Pi-hole Dashboard
    - Proxmox Web UI
    - Additional Services
```

**Key Design Decisions:**
- **DNS-01 Challenge**: Enables certificates for private services
- **File-Based Config**: Explicit, version-controllable, educational
- **Edge Node Architecture**: Production-ready pattern
- **1Password Integration**: Enterprise secrets management

---

## 🛠️ Tech Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| Traefik | v3.2 | Reverse proxy & edge router |
| Docker | 29.x+ | Container runtime |
| Let's Encrypt | - | Certificate authority |
| Cloudflare | - | DNS provider & API |
| 1Password | CLI v2.x | Secrets management |
| Pi-hole | Latest | Local DNS resolution |

**Infrastructure:**
- Raspberry Pi 5 (4GB) - Traefik edge node
- Raspberry Pi 5 - Pi-hole DNS server  
- Mini PC - Proxmox hypervisor

---

## 🚀 Quick Start

### Prerequisites
```bash
# Required
- Domain name with Cloudflare DNS
- 1Password account (Family/Teams/Business)
- Docker & Docker Compose installed
- Basic Linux/networking knowledge

# Optional but recommended
- Pi-hole for local DNS
- Static IPs for all servers
```

### Installation

**1. Clone Repository**
```bash
git clone https://github.com/marcus-singleton/homelab-infrastructure
cd homelab-infrastructure/traefik-reverse-proxy
```

**2. Configure Secrets**
```bash
# Install 1Password CLI, store the Cloudflare API token in 1Password,
# create a Service Account, and configure the op CLI locally.
```

**3. Customize Configuration**
```bash
# Copy example files
cp config/static/traefik.yml.example config/static/traefik.yml
cp docker-compose.yml.example docker-compose.yml
cp .env.example .env

# Edit files and replace:
# - yourdomain.com → your actual domain
# - your-email@example.com → your email
# - 192.168.1.x → your network IPs
# - Generate htpasswd hash for dashboard
```

**4. Start Traefik**
```bash
# Load secrets from 1Password and start
CF_DNS_API_TOKEN=$(op read "op://Traefik/Cloudflare API Token/password") \
  docker compose up -d

# Check logs
docker logs traefik -f
```

**5. Verify**
```bash
# Check certificate was obtained
cat certs/acme.json | jq '.cloudflare.Certificates[] | .domain.main'

# Access dashboard
# https://traefik.yourdomain.com
```

---

## 📚 Documentation

*Planned — not yet written. Sections below cover the same ground inline in the meantime (Quick Start, Common Issues, Adding New Services, Security).*

| Document | Description |
|----------|-------------|
| Prerequisites | Domain, DNS, 1Password setup |
| Installation Guide | Complete step-by-step walkthrough |
| Configuration | Detailed config explanations |
| Adding Services | How to add new services |
| Troubleshooting | Common issues & solutions |
| Maintenance | Day-2 operations guide |
| Architecture | Design decisions & diagrams |
| Security | Security best practices |

---

## 🎓 With This Project, I Practiced...

**Technical Skills:**
- TLS/SSL certificate lifecycle management
- DNS-01 challenge vs HTTP-01 challenge
- Docker Compose environment variable injection
- API integration (Cloudflare, 1Password)
- Reverse proxy routing patterns
- Log analysis and debugging

**DevOps Practices:**
- Infrastructure as Code principles
- Secrets management best practices
- Documentation as part of development
- Troubleshooting methodologies
- Production-ready architecture

**Problem Solving:**
- DNS resolution debugging
- Certificate issuance troubleshooting
- Browser caching issues
- Environment variable precedence
- Docker API compatibility

---

## 🐛 Common Issues

**Quick Fixes:**

| Issue | Quick Fix |
|-------|-----------|
| No certificate issued | Check CF_DNS_API_TOKEN is set |
| DNS resolving wrong IP | Update local DNS to point to Traefik |
| Browser shows "Not Secure" | Clear browser cache, verify cert exists |
| Can't access dashboard | Check BasicAuth credentials |

---

## 🚀 Adding New Services

**Quick Template:**

```yaml
# config/dynamic/myservice.yml
http:
  routers:
    myservice:
      rule: "Host(`myservice.yourdomain.com`)"
      service: myservice-service
      entryPoints:
        - https
      tls:
        certResolver: cloudflare

  services:
    myservice-service:
      loadBalancer:
        servers:
          - url: "http://192.168.1.x:port"
```

1. Create file in `config/dynamic/`
2. Add DNS record (points to Traefik)
3. Wait 2 seconds (auto-reload)
4. Access via HTTPS

---

## 🔐 Security

**Implemented:**
- ✅ No hardcoded secrets
- ✅ TLS 1.2+ only
- ✅ BasicAuth on dashboard
- ✅ Certificate auto-renewal
- ✅ HTTP → HTTPS redirect
- ✅ Container hardening

**Recommended Additions:**
- [ ] Rate limiting middleware
- [ ] SSO with Authelia/Authentik
- [ ] WAF rules
- [ ] Monitoring & alerting
- [ ] Regular security audits

---

## 📊 Metrics & Monitoring

**Current Setup:**
- Traefik Dashboard: Real-time metrics
- Log Files: `/opt/traefik/logs/`
- Certificate Status: `acme.json`

**Future Enhancements:**
- Prometheus for metrics collection
- Grafana for visualization
- Loki for log aggregation
- Alertmanager for notifications

---

## 🔄 Maintenance

**Daily:** Automated (certificates, routing)

**Weekly:**
```bash
# Check logs for errors
docker logs traefik | grep -i error

# Verify certificates valid
cat certs/acme.json | jq '.cloudflare.Certificates[] | .domain'
```

**Monthly:**
```bash
# Update Traefik
docker compose pull
CF_DNS_API_TOKEN=$(op read "op://...") docker compose up -d

# Backup configuration
```

---

## 🎯 Future Improvements

**Short-term:**
- [ ] Add more services
- [ ] Implement middleware library
- [ ] Set up monitoring stack

**Long-term:**
- [ ] Kubernetes migration
- [ ] Service mesh integration
- [ ] Multi-region setup

---

## 💼 Professional Context

**Built as part of my DevOps learning journey.**

This project demonstrates:
- Production-ready architecture patterns
- Security best practices
- Documentation skills
- Problem-solving ability
- Continuous learning mindset

See my [complete portfolio](https://github.com/marcus-singleton) for more projects.

---

<div align="center">

**🔗 [View Project](https://github.com/marcus-singleton/homelab-infrastructure) • [Connect](https://linkedin.com/in/msingleton18)**

</div>
