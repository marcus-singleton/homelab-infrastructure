# Kubernetes Cluster

**Status: Planned / in progress.** CKA passed 2026-07-16, which was the gate for this work — build starts now. This README describes the design; check back for implementation as it lands.

## Goal

Run a K3s cluster on Proxmox with Flux CD managing deployments via GitOps — Git as the control plane, not manual `kubectl apply`.

## Why Flux over ArgoCD

Decided in favor of Flux CD: lighter operational footprint, CLI-first workflow (consistent with the CKA skill set), and clean bootstrap into this repo as a monorepo. ArgoCD's UI-first model was the alternative considered but not chosen — no visual dashboard requirement here that would justify the heavier install.

## Planned Architecture

- K3s deployed on Proxmox
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
