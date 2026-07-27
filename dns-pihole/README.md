# Local DNS with Pi-hole + Unbound

<div align="center">

![Pi-hole](https://img.shields.io/badge/Pi--hole-DNS_Sinkhole-red?logo=pihole)
![Unbound](https://img.shields.io/badge/Unbound-Recursive_Resolver-blue)
![Raspberry Pi](https://img.shields.io/badge/Raspberry_Pi-5-C51A4A?logo=raspberrypi)
![Status](https://img.shields.io/badge/Status-Production-success)

*Privacy-first, ad-blocking DNS for the home network — no third-party resolver in the loop*

[Features](#-features) • [Quick Start](#-quick-start) • [Architecture](#-architecture)

</div>


## 🎯 Project Overview

This project implements network-wide DNS with:
- ✅ Ad/tracker blocking at the network level
- ✅ Full recursive resolution — no public DNS provider ever sees a query
- ✅ DNSSEC validation on every lookup
- ✅ VLAN-scoped blast radius containment
- ✅ Automated, security-focused blocklists

### The Problem

Default home DNS setups typically present:
- ❌ Every query visible to the ISP or a public resolver (Cloudflare, Google)
- ❌ No ad/tracker blocking at the network level
- ❌ A single DNS misconfiguration capable of taking down every VLAN at once

### The Solution

A dedicated DNS host that:
- Resolves recursively straight to root/TLD/authoritative servers — no third party in the loop, ever
- Validates every response with DNSSEC
- Is advertised only to the Home VLAN, containing blast radius from other network segments
- Layers security-focused blocklists (malware C2, phishing, trackers) on top of the default list


## ✨ Features

### Privacy
- 🔒 **Recursive resolution** — zero third-party DNS provider in the query path
- 🛡️ **DNSSEC validation** — every response cryptographically checked
- 🕵️ **Query minimization** — reduces what's exposed to upstream servers

### Reliability
- 🎯 **VLAN-scoped** — DNS failure or misconfiguration stays contained to the Home VLAN
- 📦 **Config backups** — Teleporter export, stored off-device

### Operations
- 📝 **Query logging** for troubleshooting
- 🔍 **Layered verification** — local Pi-hole test, then direct Unbound test, isolates which layer failed


## 🏗️ Architecture

```
[Client on Home VLAN]
         ↓
   [Pi-hole (<PI_IP>)]
         ↓
  [Unbound (127.0.0.1:5335)]
         ↓
[Root / TLD / Authoritative Servers]
```

**Key Design Decisions:**
- **Recursive resolver (Unbound), not a forwarder** — removes any third-party DNS provider from the path entirely, rather than just picking a "more private" upstream
- **VLAN-scoped advertisement** — other VLANs (IoT, Work, Lab, Management) intentionally use separate DNS, containing blast radius by design
- **Dedicated host** — DNS doesn't share a Pi with any other service


## 🛠️ Tech Stack

| Component | Purpose |
|-----------|---------|
| Pi-hole | DNS sinkhole / ad-blocker, admin UI, query log |
| Unbound | Local recursive, DNSSEC-validating resolver |
| Raspberry Pi 5 (8GB) | Dedicated DNS host, wired Ethernet |
| UniFi | VLAN definition, DHCP, DNS advertisement |


## 🚀 Quick Start

### Prerequisites
```bash
# Required
- Raspberry Pi OS (64-bit), wired Ethernet
- UniFi router/gateway with VLAN support
- A fixed IP for the Pi (UniFi DHCP reservation, not static-on-device)
```

### Installation

**1. Install Pi-hole**
```bash
ssh <PI_USER>@<PI_IP>
curl -sSL https://install.pi-hole.net | bash
# Upstream DNS during setup: any provider temporarily — replaced by Unbound next
sudo pihole -a -p   # set admin password
```

**2. Install and configure Unbound**
```bash
sudo apt install unbound -y
sudo vi /etc/unbound/unbound.conf.d/pi-hole.conf
```
```
server:
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-ip6: no
    harden-dnssec-stripped: yes
    qname-minimisation: yes
    hide-identity: yes
    hide-version: yes
    private-address: 192.168.0.0/16
```
```bash
sudo systemctl enable --now unbound
dig @127.0.0.1 -p 5335 google.com +dnssec +multi   # verify: ANSWER present, ad flag on DNSSEC pass
```

**3. Point Pi-hole at Unbound**
Pi-hole admin UI → Settings → DNS → Upstream DNS Servers → uncheck all public providers → Custom 1 (IPv4): `127.0.0.1#5335`

**4. Advertise to the Home VLAN only**
UniFi → Settings → Networks → Home VLAN → DHCP/DNS → DNS Server: `<PI_IP>`

**5. Verify from a client**
```bash
nslookup google.com   # Server should return <PI_IP>
```


## 🎓 With This Project, I Practiced

**Technical Skills:**
- Recursive vs. forwarding DNS resolution
- DNSSEC validation chains
- Network segmentation / VLAN-scoped service design
- systemd service management (`unbound`, `pihole-FTL`)

**Problem Solving:**
- Layered verification methodology — isolating which resolver stage (Pi-hole vs. Unbound) is failing before assuming the whole chain is broken


## 💼 Professional Context

**Built as part of my SRE learning journey.**

This project demonstrates network segmentation discipline, privacy-conscious infrastructure design, and layered verification under a real household uptime expectation.

See my [complete portfolio](https://github.com/marcus-singleton) for more projects.


<div align="center">

**🔗 [View Project](https://github.com/marcus-singleton/homelab-infrastructure) • [Connect](https://linkedin.com/in/msingleton18)**

</div>
