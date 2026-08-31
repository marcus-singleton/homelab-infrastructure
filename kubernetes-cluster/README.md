# Kubernetes Cluster

**Status: Cluster live, GitOps layer planned.** 3-node K3s cluster running on Proxmox — built, verified, and survived a full host reboot unattended. Flux CD GitOps on top is the next layer, not yet started. CKA (passed 2026-07-16) was the entry gate that validated readiness for this work.

## Goal

Run a K3s cluster on Proxmox with Flux CD managing deployments via GitOps — Git as the control plane, not manual `kubectl apply`.

## Cluster (Live)

| Node | Role | vCPU | RAM | IP |
|---|---|---|---|---|
| `k3s-control` | control-plane | 2 | 4 GB | `<LAB_VLAN>.201` |
| `k3s-worker1` | worker | 2 | 4 GB | `<LAB_VLAN>.202` |
| `k3s-worker2` | worker | 2 | 4 GB | `<LAB_VLAN>.203` |

Ubuntu 24.04 LTS on each node, K3s `v1.36.4+k3s1` (pinned, not an unversioned `curl \| sh`), single embedded-SQLite server (not HA — see the note in the setup runbook on what that means and how to change it later).

```
$ kubectl get nodes
NAME          STATUS   ROLES           AGE   VERSION
k3s-control   Ready    control-plane   62m   v1.36.4+k3s1
k3s-worker1   Ready    worker          62m   v1.36.4+k3s1
k3s-worker2   Ready    worker          62m   v1.36.4+k3s1
```

**Verified, not just "up":** a test deployment scheduled across all 3 nodes and served traffic through a NodePort; a full reboot of the Proxmox host brought all 3 VMs back up unattended with the cluster returning to `Ready` with no manual intervention.

## Why Flux over ArgoCD

Decided in favor of Flux CD: lighter operational footprint, CLI-first workflow (consistent with the CKA skill set), and clean bootstrap into this repo as a monorepo. ArgoCD's UI-first model was the alternative considered but not chosen — no visual dashboard requirement here that would justify the heavier install.

## Planned: The GitOps Layer

- Flux bootstrapped against this GitHub repo (`main` branch, 1m poll interval)
- First app deployed via GitOps reconciliation
- Flux sync-status surfaced as a Grafana panel (ties into `../monitoring-stack`)

Bootstrap config (planned):

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: homelab-infrastructure
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/marcus-singleton/homelab-infrastructure
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: homelab-apps
  namespace: flux-system
spec:
  interval: 5m
  path: ./k3s/apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: homelab-infrastructure
```

## Planned: The Intentional Incident

Once Flux is live and reconciling, deliberately break a deployed resource, observe Flux detect the drift and reconcile it, and document the whole thing as a blameless postmortem. That exercise — not a clean deploy — is the actual portfolio artifact: it demonstrates GitOps self-healing in practice, not just in theory.

## Result (when complete)

Every Kubernetes workload change will be auditable, reversible, and reconciled automatically from Git — the same discipline expected of production Kubernetes operations.
