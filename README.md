# kahal

Homelab Kubernetes, managed with Flux CD. Everything is declared in Git, secrets are encrypted with SOPS, and services are reachable only over Tailscale.

## Stack

| Area | Choice |
| --- | --- |
| Kubernetes | k3s on a single bare-metal server (`emet`) |
| GitOps | Flux CD |
| Config | Kustomize (`base/` + `staging/` overlays) and Helm |
| Secrets | SOPS with Age |
| Access | Tailscale Kubernetes operator (no public ingress) |
| Storage | NFS-backed PVCs |
| Databases | CloudNativePG with Barman Cloud backups |
| Monitoring | kube-prometheus-stack (Prometheus + Grafana) |
| Updates | Renovate |

## Services

| Service | Tailscale hostname | Port |
| --- | --- | --- |
| Linkding (bookmarks) | `linkding` | 9090 |
| Grafana (dashboards) | `grafana` | 3000 |

Planned: Miniflux for RSS, with Linkding as the long-term archive.

## Layout

```
apps/             # Workloads (base + staging overlay)
clusters/staging/ # Flux entrypoint and Kustomizations
infrastructure/   # Cluster controllers: cert-manager, cnpg, renovate
monitoring/       # kube-prometheus-stack, Tailscale operator
```

Each component is defined once under `base/` and wired into the cluster through a `staging/` overlay.

## Rebuilding

A rebuild needs three things: this repo, the Flux SSH deploy key, and the SOPS Age key (kept outside the repo). The full bootstrap steps, secrets workflow and known gotchas are in [ARCHITECTURE.md](ARCHITECTURE.md).

## Notes

- Changes go through Git; Flux reconciles the cluster from `main`.
- Edit encrypted files with `sops <file>`, never by hand.
