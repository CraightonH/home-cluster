# Talos v1.12 Performance Considerations

read_when: upgrading to Talos 1.12+, debugging slow container startup, modifying Image Factory schematics

## KSPP kernel hardening (`init_on_alloc=1`)

Talos 1.12 enables stricter KSPP (Kernel Self-Protection Project) defaults including `init_on_alloc=1` and `slab_nomerge`. These zero every page/slab allocation at the kernel level. Python-heavy workloads (Home Assistant) are hit hardest — HA startup regressed from <1 min to ~4 min on a Xeon X5650.

**To disable `init_on_alloc`**, the kernel arg must be baked into the **Image Factory schematic** — `machine.install.extraKernelArgs` is incompatible with UKI boot (`grubUseUKICmdline`), which Talos 1.12 enables by default.

Schematic example (amd64):
```yaml
customization:
  extraKernelArgs:
    - init_on_alloc=0
  systemExtensions:
    officialExtensions:
      - siderolabs/amdgpu-firmware
      - siderolabs/iscsi-tools
      - siderolabs/util-linux-tools
```

Submit to `POST https://factory.talos.dev/schematics`, get new ID, update `talosImageURL` in talconfig.yaml, then `talosctl upgrade` each node to the new image (same Talos version, new schematic).

**Security tradeoff**: `init_on_alloc=0` removes kernel heap zeroing on allocation. Exploitation requires a kernel vuln or unsigned module (blocked by `module.sig_enforce=1`). Low risk for home/trusted workloads; not recommended for multi-tenant.

## `grubUseUKICmdline` on upgraded nodes

talhelper 3.x generates `machine.install.grubUseUKICmdline: true` for Talos 1.12 configs. **This field is rejected by nodes upgraded from 1.11** — only works on fresh installs. After `talhelper genconfig`, strip this line from each `home-node*.yaml` before applying, or the node enters maintenance mode with invalid config.

Recovery: `talosctl apply-config --insecure -n <ip> -f <fixed-config>` from maintenance mode.

## `disable-admission-controller.yaml` JSON6902 incompatibility

Talos 1.12 multi-document machine config does not support JSON6902 patches. The existing `- op: remove` patch breaks `talhelper genconfig`. Convert to strategic merge:

```yaml
# Before (broken with Talos 1.12 multi-doc)
- op: remove
  path: /cluster/apiServer/admissionControl

# After (works)
cluster:
  apiServer:
    admissionControl: []
```

Not yet applied — the JSON6902 patch still works with pre-generated configs applied via `talosctl apply-config`; it only breaks `talhelper genconfig`.

## TODO

- [ ] Create new schematics with `init_on_alloc=0` and upgrade all nodes
- [ ] Convert admission controller patch to strategic merge
- [ ] Test `slab_nomerge=0` if `init_on_alloc=0` alone is insufficient
