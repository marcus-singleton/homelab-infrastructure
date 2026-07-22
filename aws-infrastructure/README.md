# AWS Infrastructure — Backups & Cloud Integration

**Status: Planned / in progress.** S3 bucket and IAM user provisioning are the next open task — nothing is live yet.

## Goal

Offsite backup for the homelab: Proxmox VM/CT snapshots, homelab config state, and household files (family docs, Time Machine) all land in S3 via Restic, so a total on-prem hardware failure doesn't mean total data loss.

## Planned Architecture

- **Restic** as the backup engine, repository target: S3
- **IAM user** scoped to a single backup bucket (least-privilege, not an account-wide key)
- **1Password** stores the AWS credentials — pulled at runtime via the CLI (`op read`), never hardcoded or committed
- **Prometheus** textfile metric records last successful backup job
- **Grafana** alerts if the last successful backup is more than 25 hours old

## Backup Sources (planned)

| Source | Path today | Destination |
|---|---|---|
| Proxmox VM/CT | `proxmox-backups` NFS share on UNAS 4 | → Restic → S3 |
| Homelab config snapshots | `homelab-configs` NFS share | → Restic → S3 |
| Household family docs | `family-docs` SMB share | → Restic → S3 |

## Why This Matters

Zero-trust secrets handling extends to cloud credentials, not just application secrets — the AWS keys never touch a config file or shell history, only 1Password at invocation time. Backup success is a monitored SLO, not a "set it and forget it" cron job: if Restic silently stops working, Grafana pages before the data's actually gone.

## Result (when complete)

Full offsite recovery path for every piece of homelab and household data, with backup health as a first-class monitored metric rather than an assumption.
