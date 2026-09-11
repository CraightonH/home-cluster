# OpenClaw revival on the home cluster

Date: 2026-09-11. Status: draft for review.

## Goal

Bring the `agents/clawdbot` deployment back as OpenClaw 2026.9.x, driven by a ChatGPT
subscription instead of Anthropic, while keeping the agent's existing personality and
knowledge files. Conversation history from the old install is not a goal.

## Findings that shape the design

- The old install was already upstream OpenClaw 2026.3.22, wrapped by the private image
  repo `CraightonH/clawdbot-k8s`. The workspace (`/home/node/clawd`) is a git checkout of
  that repo and is in sync with GitHub at `90044c5`.
- Since 2026.3.22, `CLAWDBOT_*` env vars are silently ignored, auth profiles and sessions
  live in SQLite, `TOOLS.md` and `HEARTBEAT.md` are retired, and health endpoints are
  `/healthz`, `/startupz`, `/readyz`.
- The gateway lock uses hard links and state is WAL-mode SQLite. Upstream says not to put
  `~/.openclaw` on NFS (openclaw/openclaw#81089, unresolved). The current PVCs are NFS.
- OpenClaw's `openai` provider covers both API keys and ChatGPT subscription OAuth. A
  headless pod logs in with a device code. Copying tokens from a laptop is discouraged
  because refresh tokens get invalidated.
- The custom image bakes in kubectl, doppler, gh, yq, flux, talosctl, and claude. It copies
  no workspace files, so the public Docker Hub image leaks nothing. It ends as `root`
  because `USER node` is commented out.

## Approach

Fresh runtime state, adopted workspace. The old NFS volumes are left untouched as the
rollback.

### Storage

| Volume | Class | Mount | Purpose |
|---|---|---|---|
| `openclaw-home` (new, 10Gi, RWO) | `synology-iscsi` | `/home/node` | Whole home: `.openclaw` state, dotfiles, npm cache |
| `clawdbot-workspace` (existing, 10Gi) | `nfs-client` | `/home/node/clawd` via subPath `clawd` | Markdown workspace, git checkout |

OpenClaw treats `~` as home, and the old layout persisted all of `/home/node`. The new
block volume keeps that behaviour so `.talos`, `.gitconfig`, `.doppler`, and the npm
cache survive restarts, while the SQLite state stays off NFS. The workspace PVC is
mounted on top with `subPath: clawd`, which also hides the stale `.openclaw/` on NFS.

One-time seed: before the first boot, a job mounts the old NFS home read-only and copies
`.talos/config`, `.gitconfig`, `.doppler`, and `.config/sops` into the new home, then
chowns the new home to 1000:1000. Not copied, on purpose: `.openclaw` (fresh state),
`.claude`, `.cargo`, `.rustup`, `.npm`, `.kube/cache`.

Backups: `openclaw backup create` on a schedule is out of scope for the first pass. The
workspace is already backed up by being a git repo. State is recreatable by logging in
again; the seeded dotfiles are recoverable from the old NFS volume until it is deleted.

### Configuration

The old `openclaw.json` on the NFS volume is carried over, not rewritten in git. On
first boot the init container copies `/oldhome/.openclaw/openclaw.json` and
`/oldhome/.openclaw/credentials/` (Discord allowlist and pairing state) onto the new
home PVC, then strips the keys that are dead or would block the OpenAI route:

```
jq 'del(.auth, .agents.defaults.model, .agents.defaults.models, .agents.defaults.contextPruning)'
```

- `auth`: Anthropic profile, no longer usable.
- `agents.defaults.model` and `.models`: pinned to `sonnet` with an Anthropic-only
  allowlist; the default is set after login with `openclaw models set`.
- `agents.defaults.contextPruning`: not confirmed to exist in the 2026.9 schema, and
  the schema is strict, so it goes rather than risk a refused start.

Everything else (Discord guild allowlist, ack reaction, gateway bind and token mode,
heartbeat every 3h, concurrency) carries over unchanged. `openclaw doctor --fix` runs
once after boot to normalize any renamed keys. `OPENCLAW_CONFIG_READONLY=1` is not used;
the agent edits its own config.

Risk: if another key has since been removed from the schema, the gateway logs a schema
error and refuses to start. Fix is a `jq` delete inside the pod.

### Secrets

Doppler `home/main` is already synced wholesale into `clawdbot-secret`. Changes:

- Add `OPENCLAW_GATEWAY_TOKEN` (new name; the `CLAWDBOT_GATEWAY_TOKEN` value can be reused).
- Nothing is added for ChatGPT. The OAuth refresh token lives in the state PVC after a
  one-time `openclaw models auth login --provider openai --device-code` run inside the pod.
- Optional fallback: `OPENAI_API_KEY`, only if the subscription route is cut off.

The Doppler CLI on this laptop is logged into the work account and cannot see the home
project, so the token is added through the Doppler UI.

### Image

Keep the custom image. Reinstalling six CLIs on every restart is slow and the pod restarts
whenever Doppler changes (reloader). Changes in `clawdbot-k8s`:

- Merge Renovate PR #29 (base 2026.3.22 to 2026.9.4). Land it via a PR that also carries
  the edits below so CI builds once.
- Drop the `claude` install. It cannot run on a subscription anymore and it installs into
  `/root`. The old `claude -p` delegation pattern is now OpenClaw's ACP feature
  (`acp.enabled`, `plugins.entries.acpx`), which fetches the Claude, Codex, or Gemini
  bridges via npx on first use. Enabling it later is a config change, not an image change.
  Claude via ACP would need an Anthropic API key, not a subscription.
- Do not install the Codex CLI. Long coding turns route through OpenClaw's native Codex
  harness (`agentRuntime.id: "codex"` on the OpenAI model), which reuses the `openai`
  OAuth profile from the device-code login.
- ACP bridges fetched by npx cache into `~/.npm`, which persists because the whole home
  is on the block PVC. No image change needed if ACP is enabled later.
- Restore `USER node` at the end. Upstream runs as UID 1000 and the k8s manifest sets
  `runAsUser: 1000`, `fsGroup: 1000`. The new iSCSI PVC starts empty so ownership is
  clean. The NFS workspace files are owned by root; a one-off `chown -R 1000:1000` job on
  the workspace subpath runs before the first boot.
- Bump the kubectl apt repo from 1.31 to 1.34 to match the cluster.
- Renovate PRs 25 to 28 and 30 (GitHub Actions majors) are merged if CI passes.

### Deployment changes (`kubernetes/apps/agents/clawdbot/app/helmrelease.yaml`)

- Image tag `2026.9.4-k8s`.
- Env: `OPENCLAW_STATE_DIR=/home/node/.openclaw`, `OPENCLAW_GATEWAY_PORT=18789`,
  `HOME`, `TZ`, `TERM` unchanged. Remove all `CLAWDBOT_*` and `OPENCLAW_ALLOW_MULTI_GATEWAY`.
- Command: `node dist/index.js gateway --bind lan --port 18789`. Drop the lock-file
  cleanup and `startup-wake.sh` from the command. Lock cleanup is unnecessary on block
  storage. Startup wake is revisited once heartbeat is confirmed working under 2026.9.
- Probes: startup `/startupz` (30 x 10s), liveness `/healthz`, readiness `/startupz`.
- `strategy: Recreate` so two gateways never share the state PVC.
- Security context: `runAsUser/runAsGroup/fsGroup 1000`, drop all capabilities. Root
  filesystem stays writable because the image's tool installs expect it.
- `replicas: 1` explicitly, since the current deployment was hand-scaled to 0.
- Persistence block rewritten per the storage table. The `clawdbot-config` PVC reference
  stays out; that PVC is deleted by hand after the new install is stable.
- Rename the release, service, and ingress from `clawdbot` to `openclaw` is deferred. Names
  stay as they are to keep the diff reviewable; the ingress host `clawd.hancockfam.net`
  stays.

### Workspace adjustments (in `clawdbot-k8s`, done by `doctor --fix` or by hand)

- `TOOLS.md` content merges into a `## Tools` section of `AGENTS.md`; the file is removed.
- `HEARTBEAT.md` startup-check logic moves to `AGENTS.md`; the file is removed. The
  pending-restart pattern still works because it only depends on `memory/`.
- `AGENTS.md` references to `node /app/dist/index.js models status` and 5-hour Anthropic
  usage blocks are rewritten or removed; they do not apply to the OpenAI route.
- `USER.md` and `IDENTITY.md` unchanged.
- Delete `memory/.ha-db-password` from the NAS copy. It is gitignored but sits in
  plaintext on the volume.

### First-boot runbook

1. Merge image PR, wait for `craighton/clawdbot-k8s:2026.9.4-k8s`.
2. Add `OPENCLAW_GATEWAY_TOKEN` to Doppler.
3. Merge the home-cluster PR. Flux creates the PVC, runs the chown job, starts the pod.
4. `kubectl exec` into the pod, run `openclaw models auth login --provider openai
   --device-code`, approve in a browser.
5. `openclaw models list --provider openai`, then `openclaw models set <id>` if the seeded
   default is not available on the account.
6. `openclaw doctor --fix` with the gateway running is acceptable here because there is no
   legacy session history to import; it handles the TOOLS/HEARTBEAT merge.
7. Verify: `/healthz` 200, Discord mention gets a reply, `kubectl get pods -A` works from
   inside the pod, a heartbeat fires within 3h.

### Rollback

Revert the home-cluster PR. The old NFS PVCs and the `2026.3.22-k8s` image are unchanged.
The new iSCSI PVC has `retain: true` and is deleted by hand if the revival is abandoned.

## Risks

- OpenAI policy on subscription use in third-party harnesses is asserted by OpenClaw's
  docs without a cited OpenAI source. The API-key fallback exists for this reason.
- `synology-iscsi` volumes have gone read-only after NAS blips before. The state dir is
  recreatable, but a wedged SQLite would need the pod recycled.
- Running as UID 1000 changes file ownership on the shared NFS workspace; anything else
  that reads that PVC as root keeps working, but writes from other UIDs would fail.

## Out of scope

Migrating old sessions, renaming the release to `openclaw`, scheduled backups, Control UI
over the ingress without device pairing.
