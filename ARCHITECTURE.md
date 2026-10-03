# kahal: architecture and context

Homelab Kubernetes cluster managed with Flux CD (GitOps). This file is meant to be read by humans and by Claude (as Project knowledge or as `CLAUDE.md`). Items marked **TODO** still need checking.

## Stack at a glance

| Area | Choice |
| --- | --- |
| Distribution | k3s |
| GitOps | Flux CD, repo `jeff-cevaal/kahal`, branch `main` |
| Config layering | Kustomize (`base/` + `staging/` overlays) + Helm via HelmRelease |
| Secrets | SOPS with Age |
| Dependency updates | Renovate (`renovate.json`, plus a `renovate` controller under infrastructure) |
| Remote access | Tailscale Kubernetes operator (no public ingress) |
| Storage | NFS-backed PVCs (standard pattern) |
| Databases | CloudNativePG (cnpg), backups via Barman Cloud plugin |
| Monitoring | kube-prometheus-stack (Prometheus + Grafana) |
| Tooling | k9s, kubectl, flux CLI |

## Target

The cluster runs on a single bare-metal server, **`emet`**. The earlier VMs (`tzeil-vm`, `emet-vm`) have been removed, so there are no per-target overlays and no hostname suffixes. Services use plain Tailscale names (`grafana`, `linkding`). The only overlay layer is `staging/`.

## Repository layout

```
kahal/
├── apps/
│   ├── base/linkding/          # cnpg-database, configmap, deployment, kustomization,
│   │                           #   namespace, secrets, service, storage
│   └── staging/linkding/       # kustomization.yaml (overlay on base)
├── clusters/staging/           # Flux entrypoint
│   ├── flux-system/            # gotk-components, gotk-sync, kustomization, secrets
│   ├── infrastructure.yaml     # Kustomization `infrastructure-controllers` -> infrastructure/controllers/staging
│   ├── monitoring.yaml         # Kustomization `monitoring-controllers` -> monitoring/controllers/staging
│   └── apps.yaml               # Kustomization `apps` -> apps/staging
├── infrastructure/controllers/
│   ├── base/                   # cert-manager, cnpg, renovate
│   └── staging/                # overlays for the same three
├── monitoring/controllers/
│   ├── base/                   # kube-prometheus-stack, tailscale-operator
│   └── staging/                # overlays for the same two
├── .sops.yaml                  # SOPS creation rules (Age recipient, encrypted_regex)
├── .gitignore
├── renovate.json
└── README.md
```

Pattern: every component has a `base/` definition and a `staging/` overlay, and the Flux Kustomizations in `clusters/staging/` point at the `staging/` directories.

## Reconcile order

All three Flux Kustomizations use `prune: true`, SOPS decryption (`sops-age` secret), and a 1m interval.

- `apps` **dependsOn** `infrastructure-controllers`, so cnpg (and the operators Linkding relies on) exist before Linkding's database objects apply.
- `infrastructure-controllers` sets `wait: true`, so it only reports Ready once its Helm releases are healthy, and `apps` waits for that.
- `monitoring-controllers` has no `dependsOn` yet.

`dependsOn` waits for the upstream Kustomization to be Ready. Without `wait: true` on it, Ready only means the manifests were applied, not that the Helm releases are healthy.

**Planned:** move the Tailscale operator from `monitoring/controllers` to `infrastructure/controllers`, since it's cluster plumbing that both Grafana and Linkding rely on. Once moved, `monitoring-controllers` should `dependsOn` `infrastructure-controllers` (Grafana's Tailscale annotations need the operator). Moving a HelmRelease between two pruning Kustomizations can briefly uninstall it; see the `kustomize.toolkit.fluxcd.io/prune: disabled` annotation if that matters.

## Services

| Service | Tailscale hostname | Port | Notes |
| --- | --- | --- | --- |
| Linkding | `linkding` | 9090 | Bookmark manager. CNPG database backed up via Barman Cloud. |
| Grafana | `grafana` | 3000 | Exposed via the Tailscale operator, configured with annotations in the HelmRelease values. |

Planned: Miniflux (RSS aggregator), with Linkding as the long-term archive.

## Secrets (SOPS / Age)

- Secrets are SOPS-encrypted with Age and committed to the repo; Flux decrypts them in-cluster. Examples: `apps/base/linkding/secrets.yaml`, `clusters/staging/flux-system/secrets.yaml`.
- The Age private key is **not** in the repo. It must be placed in the cluster manually on a rebuild. This has been a recurring pain point.
- The Age key lives on the desktop (where it originated) and on `emet`, outside the repo. Keep an off-machine backup too; losing it makes every encrypted secret unrecoverable.
- `.sops.yaml` in the repo root sets the Age recipient and `encrypted_regex: ^(data|stringData)$`, so only secret values are encrypted and `metadata` stays readable.
- Edit secrets with `sops <file>` in a terminal, not in an editor with format-on-save. The MAC covers the plaintext fields too, so changing a name, namespace or type by hand after encrypting causes a "MAC mismatch" on decrypt.
- Never commit plaintext secrets or the Age key.

## Bootstrap

Flux authenticates to GitHub over **SSH** (deploy key), not HTTPS.

1. `flux bootstrap git` with `--path=clusters/staging`.
2. Create the Age key secret for SOPS.
3. The git credentials for flux-system live in `clusters/staging/flux-system/secrets.yaml`: the SOPS-encrypted SSH deploy key (public and private key) plus GitHub's `known_hosts`. Flux applies it from the repo once the Age secret exists, so the Age key is the only credential held outside Git.

## Operational notes and gotchas

- **firewalld on AlmaLinux:** add the CNI interfaces `cni0` and `flannel.1` to the trusted zone, otherwise kube-prometheus-stack scrape targets break.
- **DNS on AlmaLinux:** Tailscale DNS conflicted with CoreDNS. Fixed with systemd-resolved plus the k3s `resolv-conf` setting.
- **Tailscale operator OAuth:** if the operator fails to authenticate, check the OAuth client scopes first, then annotation placement in the HelmRelease values.
- **SOPS "MAC mismatch":** the key is fine (a wrong key fails earlier); the file was modified after encryption. Fix: `sops -d --ignore-mac -i <file>`, check the plaintext, then `sops -e -i <file>`.
- **cert-manager:** exists in `infrastructure/controllers`, even though everything is reached over Tailscale. **TODO:** note why it's there (likely a cnpg/Barman Cloud plugin dependency).

## History

1. Originally `tzeil-cluster` on Ubuntu; migrated to `kahal` on AlmaLinux.
2. Migration fixes: Flux bootstrap `--path` misconfiguration, CoreDNS/Tailscale DNS conflict, SOPS Age key management.
3. Set up the Tailscale operator to expose Grafana (OAuth scope and annotation debugging).
4. Deployed Linkding; ran across `tzeil-vm`, then `emet-vm`, then removed both VMs and consolidated onto bare-metal `emet`.
5. Cnpg on `emet`, with scheduled Linkding backups via Barman Cloud.
6. Removed a stray `Projects/clusters/staging/flux-system` directory: a copy of the Flux components that Renovate had been bumping (duplicate `chore(deps)` PRs). `clusters/staging` is the only Flux path.

## Conventions

- Everything goes through Git; avoid `kubectl apply` for anything Flux manages.
- `base/` holds the definition, `staging/` holds the overlay.
- NFS-backed PVCs for persistent storage.
- Validate before pushing: `kubectl kustomize <path>` or `flux build kustomization <name> --path <path>`.
