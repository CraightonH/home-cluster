# Cluster Upgrade Plan: Talos 1.9 / k8s 1.32 / Flux 2.7

## Progress

| Phase | Status | Date | Notes |
|-------|--------|------|-------|
| 1. Talos v1.7.5 -> v1.8.4 | **DONE** | 2026-02-08 | All 5 nodes upgraded; host-dns svc deleted; committed `7d3a47e` |
| 2. Kubernetes v1.30.2 -> v1.31.6 | **DONE** | 2026-02-08 | All 5 nodes on v1.31.6; committed `720b849` |
| 3. Rancher 2.10.3 -> 2.11.3 | **DONE** | 2026-02-08 | HelmRelease upgraded successfully; committed `1e5dec8`; rancher kustomization was suspended -- resumed it |
| 4. Talos v1.8.4 -> v1.9.6 | **DONE** | 2026-02-08 | All 5 nodes upgraded; systemd-udevd transition clean; network OK |
| 5. Flux v2.3.0 -> v2.7.5 | **DONE** | 2026-02-08 | Bootstrap + PR #64 merged; cluster kustomization unblocked; enshrouded deploying |
| 6. Kubernetes v1.31.6 -> v1.32.3 | **DONE** | 2026-02-08 | All 5 nodes on v1.32.3; clean upgrade |

**Current state:** Talos v1.9.6 / Kubernetes v1.32.3 / Rancher 2.11.3 / Flux v2.7.5

## Context

The `cluster` Flux kustomization is stuck because `flux.yaml` references `OCIRepository` at `source.toolkit.fluxcd.io/v1`, a CRD that doesn't exist in Flux v2.3.0. PR #64 upgrades Flux to v2.7.5, but that requires k8s >= 1.32. K8s 1.32 requires Talos >= 1.9 and Rancher >= 2.11. This cascading dependency means a full stack upgrade is needed to unblock Flux (and therefore the enshrouded deployment).

**Target:** Talos v1.9.6 / Kubernetes v1.32.x / Flux v2.7.5 / Rancher 2.11.x

## Version Matrix

| Component | Current | Target | Upgrade Path |
|-----------|---------|--------|-------------|
| Talos | v1.8.4 | v1.9.6 | 1.8.4 -> 1.9.6 |
| Kubernetes | v1.31.6 | v1.32.x | 1.31 -> 1.32 |
| Flux | v2.3.0 | v2.7.5 | Direct (bootstrap + PR #64) |
| Rancher | 2.11.3 | 2.11.3 | Done |

All other operators (Cilium 1.19, cert-manager v1.19.3, ingress-nginx 4.14.3, CNPG 0.27.1, external-secrets 2.0.0, etc.) already verified compatible with k8s 1.32.

## Node Inventory

| Node | IP | Role | Arch | Schematic (short) | Reboot Mode |
|------|-----|------|------|-------------------|-------------|
| node1 | .241 | control-plane | amd64 | `48369893...` | default |
| node2 | .242 | control-plane | amd64 | `48369893...` | default |
| node3 | .243 | worker | amd64 | `48369893...` | default |
| node4 | .244 | control-plane | arm64/RPi | `f8a903f1...` | **stage** (1.9+) |
| node5 | .245 | worker | arm64/RPi | `f8a903f1...` | **stage** (1.9+) |

Full schematic IDs:
- amd64: `48369893e1b56c3e2c53575db56df55a90e260135591a0456b128e3459151c7a`
- arm64: `f8a903f101ce10f686476024898734bb6b36353cc4d41f348514db9004ec0a9d`

---

## Phase 1 -- Talos v1.7.5 -> v1.8.4 -- DONE

All 5 nodes upgraded via `talosctl upgrade` (not `task`, which wasn't on PATH).
Actual starting version was v1.7.5 (not v1.7.7 as talconfig claimed).

**Execution notes:**
- node5 (arm64 worker): OK
- node3 (amd64 worker): OK
- node2 (amd64 CP): first attempt failed with etcd health timeout; retry succeeded
- node4 (arm64 CP RPi): OK, slower reboot (~60 retries)
- node1 (primary CP): OK, ~80 retries during reboot
- host-dns service deleted post-upgrade
- Commit: `7d3a47e` -- `build: bump talosVersion to v1.8.4`

---

## Phase 2 -- Kubernetes v1.30.2 -> v1.31.6 -- DONE

Upgraded via `talosctl --nodes 192.168.1.241 upgrade-k8s --to 1.31.6`.
Dry-run passed. Full upgrade completed cleanly.

- Commit: `720b849` -- `build: bump kubernetesVersion to v1.31.6`

---

## Phase 3 -- Rancher 2.10.3 -> 2.11.3 -- DONE

Bumped chart to 2.11.3 (latest stable 2.11.x). Pushed commit.

**Execution notes:**
- Rancher kustomization was `suspend: true` -- had to resume it via patch
- After resume, kustomization reconciled to latest revision and Helm upgrade ran
- Deployment rolled out successfully: `rancher@2.11.3`
- Commit: `1e5dec8` -- `build: bump rancher chart to 2.11.3`

---

## Phase 4 -- Talos v1.8.4 -> v1.9.6 -- DONE

Same pattern as Phase 1. **Key change:** RPi nodes (node4/5) use `mode=stage` for safety.

> **Note:** `task` (go-task) not on PATH in WSL. Use `talosctl` directly:
> `TALOSCONFIG=kubernetes/bootstrap/talos/clusterconfig/talosconfig talosctl --nodes <IP> upgrade --image <image> --wait=true --timeout=10m --preserve=true --reboot-mode=<default|stage>`

**Images:** same schematic IDs, tag `:v1.9.6`

**Watch for:**
- eudev -> systemd-udevd: interface names may change. Config uses MAC-based `deviceSelector` so should be safe. Verify with `talosctl get links` after first node.
- amdgpu-firmware: already in amd64 schematic as extension -- transparent.
- RPi boot: sbc-raspberrypi v0.1.1+ fixes included in current factory images.

```bash
# Workers
task talos:upgrade node=192.168.1.245 image=factory.talos.dev/installer/f8a903f101ce10f686476024898734bb6b36353cc4d41f348514db9004ec0a9d:v1.9.6 mode=stage   # node5 RPi
task talos:upgrade node=192.168.1.243 image=factory.talos.dev/installer/48369893e1b56c3e2c53575db56df55a90e260135591a0456b128e3459151c7a:v1.9.6              # node3

# Control planes
task talos:upgrade node=192.168.1.242 image=factory.talos.dev/installer/48369893e1b56c3e2c53575db56df55a90e260135591a0456b128e3459151c7a:v1.9.6              # node2
task talos:upgrade node=192.168.1.244 image=factory.talos.dev/installer/f8a903f101ce10f686476024898734bb6b36353cc4d41f348514db9004ec0a9d:v1.9.6 mode=stage   # node4 RPi
task talos:upgrade node=192.168.1.241 image=factory.talos.dev/installer/48369893e1b56c3e2c53575db56df55a90e260135591a0456b128e3459151c7a:v1.9.6              # node1
```

**Verify:** all nodes Talos v1.9.6, network connectivity intact

**Git:** update `talconfig.yaml` line 4: `talosVersion: v1.9.6`, commit + push

---

## Phase 5 -- Flux v2.3.0 -> v2.7.5

Two-step: bootstrap CRDs first (manual kubectl), then merge PR #64.

```bash
# Step 1: checkout PR branch and apply bootstrap (installs v2.7.5 CRDs + controllers)
git fetch origin renovate/flux && git checkout renovate/flux
kubectl apply --server-side --kustomize kubernetes/bootstrap/flux
git checkout main

# Verify controllers upgraded
kubectl -n flux-system get deploy -o wide   # images show v2.7.5 components

# Step 2: merge PR #64 (updates flux.yaml OCIRepository tag + bootstrap ref in git)
gh pr merge 64 --merge
git pull
```

**Verify:**
```bash
flux check                                                     # healthy, version 2.7.5
flux get sources oci                                           # flux-manifests reconciled
kubectl get kustomization -n flux-system cluster               # Ready: True (was False!)
kubectl get helmrepository -n flux-system jsknnr               # now exists -> enshrouded unblocked
```

---

## Phase 6 -- Kubernetes v1.31 -> v1.32

All prereqs met: Talos 1.9, Rancher 2.11, Flux 2.7.

```bash
task talos:upgrade-k8s controller=192.168.1.241 to=1.32.3  # or latest patch
```

**Verify:**
```bash
kubectl get nodes -o wide                                      # all v1.32.x
kubectl get pods -A | grep -v Running | grep -v Completed
flux get kustomizations                                        # all Ready
kubectl -n cattle-system get pods                              # Rancher healthy
kubectl -n games get helmrelease enshrouded                    # should be installing/ready
```

**Git:** update `talconfig.yaml` line 6: `kubernetesVersion: v1.32.3`, commit + push

---

## Post-Upgrade

1. **Verify enshrouded deploys:** `kubectl -n games get pods | grep enshrouded`
2. **Full cluster health:** `talosctl health --server=false --nodes 192.168.1.241`
3. **Close/update stale Renovate PRs** -- Talos/k8s PRs that proposed higher versions

## Risk Summary

| Phase | Risk | Mitigation |
|-------|------|-----------|
| 4 (Talos 1.9 RPi) | Highest -- eudev->systemd-udevd, RPi boot | MAC-based deviceSelector; `--stage` mode; sbc-raspberrypi v0.1.1+ |
| 5 (Flux) | Medium -- entire GitOps depends on it | Bootstrap-first approach; controllers are backwards-compatible |
| 3 (Rancher) | Low -- standard Helm bump | Easy git revert |
| 1,2,6 (Talos/k8s) | Low -- well-tested paths | `talosctl rollback` / `upgrade-k8s --to <old>` |

## Critical Files

- `kubernetes/bootstrap/talos/talconfig.yaml` -- talosVersion + kubernetesVersion (updated 4 times)
- `kubernetes/apps/cattle-system/rancher/app/helmrelease.yaml` -- chart version bump
- `kubernetes/bootstrap/flux/kustomization.yaml` -- Flux CRD bootstrap (via PR #64)
- `kubernetes/flux/config/flux.yaml` -- OCIRepository tag (via PR #64)
- `.taskfiles/Talos/Taskfile.yaml` -- task definitions (read-only, already supports `mode` param)
