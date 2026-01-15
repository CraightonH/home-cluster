# AGENTS.md - Agent Guidelines for home-cluster

This is a Kubernetes homelab cluster managed via GitOps. Changes to this repo automatically sync to the cluster.

## ⚠️ Devcontainer Required

**All cluster tooling lives in the devcontainer.** Commands like `kubectl`, `flux`, `talosctl`, `task`, `sops`, `talhelper`, and `helmfile` are only available inside the container.

```
.devcontainer/
├── devcontainer.json    # Uses ghcr.io/onedr0p/cluster-template/devcontainer:base
└── postCreateCommand.sh # Runs setup on container creation
```

**To execute cluster commands (Option 1 - Devcontainer):**
- Open the repo in VS Code and use the devcontainer
- Or run the container manually with the repo mounted

**To execute cluster commands (Option 2 - Direct Install):**
Core tools can be installed directly without the container:

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# flux CLI
curl -s https://fluxcd.io/install.sh | bash

# talosctl
curl -sL https://talos.dev/install | sh

# sops (for secrets)
curl -LO https://github.com/getsops/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64
chmod +x sops-v3.8.1.linux.amd64 && sudo mv sops-v3.8.1.linux.amd64 /usr/local/bin/sops
```

**Config files:**
```bash
export KUBECONFIG=/path/to/home-cluster/kubeconfig
export TALOSCONFIG=/path/to/home-cluster/kubernetes/bootstrap/talos/clusterconfig/talosconfig
export SOPS_AGE_KEY_FILE=/path/to/home-cluster/age.key
```

**SOPS keys in Doppler:**
The Age keys for SOPS encryption are stored in Doppler:
- `HOME_CLUSTER_SOPS_PRIVATE_KEY` - for decrypting secrets
- `HOME_CLUSTER_SOPS_PUBLIC_KEY` - for encrypting new secrets

To use: write the private key to `age.key` file or set `SOPS_AGE_KEY` env var directly.

Note: `task` commands still require the devcontainer or a local [task](https://taskfile.dev/) install with Python venv setup.

## 📦 bjw-s app-template (Default Chart)

**Unless explicitly stated otherwise, all apps use the [bjw-s app-template](https://github.com/bjw-s/helm-charts/tree/main/charts/other/app-template) chart.**

This is a generic Helm chart generator - instead of app-specific charts, you define the deployment structure in `values`. Most apps in this repo follow this pattern.

**Key resources:**
- [App-template docs](https://bjw-s.github.io/helm-charts/docs/app-template/)
- [Common library docs](https://bjw-s.github.io/helm-charts/docs/common-library/introduction/) (underlying library)
- [Values schema](https://github.com/bjw-s/helm-charts/blob/main/charts/other/app-template/values.schema.json)

**Typical HelmRelease structure:**
```yaml
spec:
  chart:
    spec:
      chart: app-template
      version: 3.2.1
      sourceRef:
        kind: HelmRepository
        name: bjw-s
        namespace: flux-system
  values:
    controllers:
      <name>:
        containers:
          <name>:
            image:
              repository: ghcr.io/example/app
              tag: v1.0.0
    service:
      <name>:
        controller: <controller-name>
        ports:
          http:
            port: 8080
    ingress:
      <name>:
        className: nginx
        hosts:
          - host: app.example.com
            paths:
              - path: /
                service:
                  identifier: <service-name>
                  port: http
    persistence:
      config:
        type: persistentVolumeClaim
        accessMode: ReadWriteOnce
        size: 1Gi
```

When asked to create or edit an app, reference existing apps in `kubernetes/apps/` for patterns and use the app-template docs for available options.

## Architecture

- **OS:** Talos Linux (immutable, API-driven)
- **GitOps:** Flux CD watches this repo and reconciles state
- **Secrets:** SOPS with Age encryption
- **Templating:** makejinja renders bootstrap config into manifests

## Directory Structure

```
kubernetes/
├── apps/           # Application manifests organized by namespace
│   ├── <namespace>/
│   │   ├── namespace.yaml
│   │   ├── kustomization.yaml
│   │   └── <app>/
│   │       ├── ks.yaml          # Flux Kustomization
│   │       └── app/
│   │           ├── helmrelease.yaml
│   │           └── kustomization.yaml
├── bootstrap/      # Initial cluster bootstrap resources
└── flux/
    ├── config/     # Flux system configuration
    ├── repositories/  # Helm/Git/OCI sources
    └── vars/       # Cluster-wide settings and secrets
```

## App Structure Pattern

Each app follows this pattern:
1. `ks.yaml` - Flux Kustomization that points to the `app/` directory
2. `app/helmrelease.yaml` - HelmRelease using typically `bjw-s/app-template` chart
3. `app/kustomization.yaml` - Kustomize resources list

Dependencies between apps are declared in `ks.yaml` via `dependsOn`.

## Common Tasks (via Taskfile)

These must be run inside the devcontainer:

```bash
task --list                    # List all available tasks
task flux:reconcile            # Force Flux to pull latest changes
task flux:apply path=<ns/app>  # Apply a specific app's Kustomization
task kubernetes:resources      # View cluster resource status
task sops:encrypt              # Encrypt all unencrypted .sops.* files
task talos:upgrade node=<n> image=<img>  # Upgrade a Talos node
```

## Security - DO NOT

- **Expose in docs/commits:** IP addresses, hostnames, MAC addresses, domain names
- **Edit directly:** `*.sops.yaml` files (use `sops` CLI to edit)
- **Commit unencrypted:** Any file matching `*.sops.*` pattern
- **Touch without understanding:** `age.key`, `admin.key`, `kubeconfig`, `talsecret.sops.yaml`

## Secrets Workflow

Inside the devcontainer:

```bash
# Edit an encrypted secret
sops kubernetes/path/to/secret.sops.yaml

# Create new secret - name it *.sops.yaml, then encrypt
task sops:encrypt
```

## Making Changes

1. **New app:** Copy existing app structure, modify HelmRelease values
2. **Update app:** Edit `helmrelease.yaml`, bump image tags or chart versions
3. **Add to namespace:** Update the namespace's `kustomization.yaml` to include the new app
4. **Commit & push:** Flux auto-reconciles (or run `task flux:reconcile` from devcontainer)

## Validation

- CI runs `kubeconform` to validate manifests
- `flux-diff` workflow shows what changes will apply
- Run locally (in devcontainer): `task kubernetes:kubeconform`

## Key Files

| File | Purpose |
|------|---------|
| `config.yaml` | Bootstrap configuration (encrypted via SOPS) |
| `kubernetes/flux/vars/cluster-settings.yaml` | Global non-secret cluster settings |
| `kubernetes/flux/vars/cluster-secrets.sops.yaml` | Global secrets (encrypted) |
| `kubernetes/flux/apps.yaml` | Root Kustomization that loads all apps |

## When Adding Apps

1. Check if HelmRepository exists in `kubernetes/flux/repositories/helm/`
2. If not, add the repo definition and update `kustomization.yaml`
3. Create app directory structure following existing patterns
4. Add app to namespace's `kustomization.yaml`
