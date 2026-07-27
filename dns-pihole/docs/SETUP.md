# DNS Setup — Pi-hole + Unbound

**Prerequisites:** Raspberry Pi OS (64-bit) image, a UniFi router/gateway with VLAN support, wired Ethernet.

## Placeholders

| Placeholder | Meaning | Example |
|---|---|---|
| `<PI_HOSTNAME>` | Raspberry Pi hostname | `pi-hole-1` |
| `<PI_USER>` | SSH username on the Pi | `pi` or `netserv` |
| `<PI_IP>` | Fixed IP of the Pi on the Home VLAN | `192.168.10.5` |
| `<HOME_VLAN_NAME>` | Name of Home network/VLAN in UniFi | `Home` |
| `<HOME_VLAN_SUBNET>` | Subnet for Home VLAN | `192.168.10.0/24` |
| `<HOME_VLAN_GATEWAY>` | Gateway/router IP for Home VLAN | `192.168.10.1` |
| `<UNIFI_CONTROLLER>` | UniFi Network UI URL | `https://unifi.example.lan:8443` |

## 1. Raspberry Pi OS and Base Setup

1. Write Raspberry Pi OS (64-bit) to microSD via Raspberry Pi Imager. In the Imager's advanced options: hostname `<PI_HOSTNAME>`, username `<PI_USER>`, SSH enabled (password auth), Wi-Fi disabled (Ethernet only).
2. Mount the Pi 5 in the rack, connect Ethernet to the UniFi switch/router.
3. First login and update:
   ```bash
   ssh <PI_USER>@<PI_IP>   # temporary DHCP IP before you fix it
   sudo apt update
   sudo apt full-upgrade -y
   sudo reboot
   ```

## 2. Static IP on the Home VLAN (UniFi Fixed IP)

Configured via UniFi, not on the Pi itself:

1. Log into UniFi Network at `<UNIFI_CONTROLLER>`.
2. Clients list → locate the Pi by hostname `<PI_HOSTNAME>` or MAC address.
3. Confirm it's on `<HOME_VLAN_NAME>` with an assigned IP.
4. Client details → **Use fixed IP** / **DHCP Reservation** → set Fixed IP to `<PI_IP>`, network `<HOME_VLAN_NAME>`.
5. Save. Confirm stable access:
   ```bash
   ssh <PI_USER>@<PI_IP>
   ```

**Result:** `<PI_IP>` is now a stable fixed IP within `<HOME_VLAN_SUBNET>`.

## 3. Pi-hole Installation

```bash
ssh <PI_USER>@<PI_IP>
curl -sSL https://install.pi-hole.net | bash
```

During interactive setup:
- **Static IP warning:** already handled via UniFi fixed IP
- **Upstream DNS:** pick any provider temporarily (e.g. Quad9, Cloudflare) — replaced by Unbound in the next step
- **Blocklists:** accept the default (StevenBlack unified list)
- **Query logging:** enable for troubleshooting, disable for stricter privacy
- **FTL privacy mode:** `0 – Show everything`

Set the admin password:
```bash
sudo pihole -a -p
```

**Local test (Pi-hole only):**
```bash
nslookup google.com 127.0.0.1
```
Expect `Server: 127.0.0.1` with a valid answer — confirms Pi-hole DNS is functional before Unbound is wired in.

## 4. Unbound Installation and Configuration

```bash
sudo apt update
sudo apt install unbound -y
sudo vi /etc/unbound/unbound.conf.d/pi-hole.conf
```

```
server:
    verbosity: 1
    interface: 127.0.0.1
    port: 5335

    do-ip4: yes
    do-ip6: no
    prefer-ip6: no

    # Security and DNSSEC
    harden-glue: yes
    harden-dnssec-stripped: yes
    harden-referral-path: yes

    # Privacy / minimal responses
    qname-minimisation: yes
    hide-identity: yes
    hide-version: yes

    # Cache sizes — conservative but ample for a Pi 5
    msg-cache-size: 50m
    rrset-cache-size: 100m

    # Prevent private ranges from leaking upstream
    private-address: 10.0.0.0/8
    private-address: 172.16.0.0/12
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 127.0.0.0/8

    num-threads: 1
    logfile: ""
    use-syslog: yes
```

> Root hints / trust anchor lines are left commented to use system defaults and reduce complexity — uncomment only if explicit management is needed later.

Enable and verify:
```bash
sudo systemctl enable unbound
sudo systemctl restart unbound
sudo systemctl status unbound   # expect: active (running)
```

**Direct Unbound test:**
```bash
dig @127.0.0.1 -p 5335 google.com +dnssec +multi
```
Expect `status: NOERROR`, an `ANSWER` section, and `ad` (Authenticated Data) in the flags when DNSSEC passes. `SERVFAIL` or timeouts here mean troubleshoot Unbound *before* integrating with Pi-hole — isolate the layer that's actually broken.

## 5. Integrate Pi-hole with Unbound

1. Pi-hole admin UI (`http://<PI_IP>/admin`) → **Settings → DNS**
2. **Upstream DNS Servers** → uncheck all public providers
3. **Custom 1 (IPv4):** `127.0.0.1#5335`
4. Save

Resolution path is now: Client → Pi-hole (`<PI_IP>`) → Unbound (`127.0.0.1:5335`) → root/TLD/authoritative servers.

## 6. UniFi: Advertise Pi-hole DNS to the Home VLAN Only

1. UniFi at `<UNIFI_CONTROLLER>` → **Settings → Networks** → select `<HOME_VLAN_NAME>` (`<HOME_VLAN_SUBNET>`)
2. **DHCP / DNS** section → **DNS Server** (or **DHCP Name Server**): `<PI_IP>`
3. Secondary DNS field: leave empty if allowed, or set to `<PI_IP>` again to prevent bypass
4. Save

**Verify from a client on `<HOME_VLAN_NAME>`:**
```bash
nslookup google.com
```
Expect `Server: <PI_IP>` with a valid answer. Cross-check in Pi-hole's **Query Log** that the client's queries appear there. Other VLANs keep whatever DNS is configured for them — unaffected by design.

## 7. Optional: Security-Focused Blocklists

Pi-hole → **Group Management → Adlists**, keep the default enabled, optionally add:

| List | URL | Purpose |
|---|---|---|
| Abuse.ch ThreatFox | `https://threatfox.abuse.ch/downloads/hostfile/` | Known malware C2/distribution domains |
| Phishing Army (Extended) | `https://phishing.army/download/phishing_army_blocklist_extended.txt` | Phishing/credential-theft domains |
| AdGuard DNS Filter | `https://adguardteam.github.io/AdGuardSDNSFilter/Filters/filter.txt` | Ads, trackers, scam/malicious domains |
| OISD Basic | `https://raw.githubusercontent.com/oisd-blocklist/oisd/main/domainswild2.list` | Curated, low-false-positive unified blocklist |

Apply:
```bash
pihole -g
```
Monitor for false positives, whitelist as needed.

## 8. Maintenance and Operations

**Regular updates:**
```bash
ssh <PI_USER>@<PI_IP>
sudo apt update
sudo apt full-upgrade -y
pihole -up
```

**Service health checks:**
```bash
sudo systemctl status unbound
sudo systemctl status pihole-FTL
```

**Backups (Teleporter):** Pi-hole admin UI → **Settings → Teleporter** → export configuration and lists → store off-device (NAS, backup drive).
