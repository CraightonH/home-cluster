# OpenClaw Revival Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the `agents/clawdbot` deployment back as OpenClaw 2026.9.4 on a ChatGPT subscription, keeping the existing Markdown workspace and starting runtime state fresh on block storage.

**Architecture:** Two repos change. `CraightonH/clawdbot-k8s` (private, GitHub) rebuilds the custom image on upstream 2026.9.4 and updates the workspace Markdown. `home-cluster` (public, GitHub, Flux-managed) rewrites the HelmRelease: whole `/home/node` on a new `synology-iscsi` PVC, the existing NFS workspace PVC mounted on top at `/home/node/clawd`, the old `openclaw.json` copied from the NFS volume by an init container on first boot, with dead Anthropic keys stripped. ChatGPT auth is a one-time device-code login inside the running pod.

**Tech Stack:** Flux v2, bjw-s app-template 3.7.3, SOPS with Age, Doppler via External Secrets, democratic-csi iSCSI, GitHub Actions, Docker Hub.

**Spec:** `docs/superpowers/specs/2026-09-11-openclaw-revival-design.md`

## Global Constraints

- Upstream image `ghcr.io/openclaw/openclaw:2026.9.4`; custom tag `craighton/clawdbot-k8s:2026.9.4-k8s`.
- Container runs as UID 1000 (`node`). No `root` at the end of the Dockerfile.
- No `CLAWDBOT_*` env vars anywhere; only `OPENCLAW_*`.
- Health endpoints: startup and readiness `/startupz`, liveness `/healthz`, port 18789.
- `home-cluster` is a **public** repo: no Discord IDs, hostnames, or tokens in plaintext. Domain via `${SECRET_DOMAIN}`. The OpenClaw config never enters git; it is copied from the NAS.
- Gateway token comes from the `OPENCLAW_GATEWAY_TOKEN` env var (already in Doppler `home/main`, synced by the existing ExternalSecret), not from the config file.
- Old NFS PVCs `clawdbot-workspace` and `clawdbot-config` are never deleted or written to by this plan except the one-time `chown` of `clawd/`.
- Work in the `home-cluster` worktree at `.worktrees/openclaw-revival` (branch `feat/openclaw-revival`). Clone `clawdbot-k8s` to `~/git/clawdbot-k8s`.
- Every kubectl call against the home cluster: `kubectl --context hancockfam-local --insecure-skip-tls-verify`. Define `k() { kubectl --context hancockfam-local --insecure-skip-tls-verify "$@"; }` at the top of each shell.
- Commit only when Craighton asks; stage and show the diff otherwise. No `Co-Authored-By` trailers.

---

## File Structure

**`~/git/clawdbot-k8s` (image + workspace repo)**
- Modify: `Dockerfile` — base 2026.9.4, remove claude, kubectl 1.34 repo, `USER node`.
- Modify: `AGENTS.md` — absorb TOOLS.md under `## Tools`, absorb HEARTBEAT.md startup check, remove Anthropic usage-block rules.
- Delete: `TOOLS.md`, `HEARTBEAT.md`, `scripts/check-claude-usage.sh`, `memory/claude-usage.md`.
- Modify: `README.md` — tool table and upstream links.

**`home-cluster` (`kubernetes/apps/agents/clawdbot/app/`)**
- Modify: `helmrelease.yaml` — full rewrite of `spec.values`.
- Delete: `secret.sops.yaml` — dead since the Doppler migration, contains a stale Anthropic key ciphertext.

---

### Task 1: Rebuild the custom image on OpenClaw 2026.9.4

**Files:**
- Modify: `~/git/clawdbot-k8s/Dockerfile`
- Modify: `~/git/clawdbot-k8s/README.md`

**Interfaces:**
- Produces: Docker Hub tag `craighton/clawdbot-k8s:2026.9.4-k8s`, entrypoint `tini -s --`, default CMD from upstream, workdir `/app`, user `node` (1000), binaries `kubectl flux talosctl gh doppler yq jq` on PATH.

- [ ] **Step 1: Clone the repo and branch**

```bash
gh repo clone CraightonH/clawdbot-k8s ~/git/clawdbot-k8s
cd ~/git/clawdbot-k8s
git checkout -b feat/openclaw-2026.9
```

- [ ] **Step 2: Edit the Dockerfile**

Replace the whole file with:

```dockerfile
# Custom OpenClaw image with k8s tools
# Base image: https://github.com/openclaw/openclaw/pkgs/container/openclaw
# Built by GitHub Actions on push; Renovate tracks the base image version.
# Workspace Markdown is NOT baked in; it lives on the NAS. This image is public.

# renovate: datasource=docker depName=ghcr.io/openclaw/openclaw
ARG OPENCLAW_VERSION=2026.9.4
FROM ghcr.io/openclaw/openclaw:${OPENCLAW_VERSION}

USER root

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
    ca-certificates curl gnupg jq git && \
    rm -rf /var/lib/apt/lists/*

# Doppler CLI
RUN curl -sLf --retry 3 --tlsv1.2 --proto "=https" \
    'https://packages.doppler.com/public/cli/gpg.DE2A7741A397C129.key' | \
    gpg --dearmor -o /usr/share/keyrings/doppler-archive-keyring.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/doppler-archive-keyring.gpg] https://packages.doppler.com/public/cli/deb/debian any-version main" \
    > /etc/apt/sources.list.d/doppler-cli.list && \
    apt-get update && apt-get install -y doppler && rm -rf /var/lib/apt/lists/*

# kubectl, matched to the cluster minor (1.34)
RUN curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | \
    gpg --dearmor -o /usr/share/keyrings/kubernetes-apt-keyring.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
    > /etc/apt/sources.list.d/kubernetes.list && \
    apt-get update && apt-get install -y kubectl && rm -rf /var/lib/apt/lists/*

# yq
RUN curl -sL https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -o /usr/local/bin/yq && \
    chmod +x /usr/local/bin/yq

# GitHub CLI
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | \
    dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg && \
    chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
    > /etc/apt/sources.list.d/github-cli.list && \
    apt-get update && apt-get install -y gh && rm -rf /var/lib/apt/lists/*

# Flux CLI
RUN curl -s https://fluxcd.io/install.sh | bash

# talosctl
RUN curl -sL https://talos.dev/install | sh

# Upstream runs as node (uid 1000). The PVC at /home/node is chowned by the
# deployment's init container, so nothing here needs to own it.
USER node
```

Note: `claude` is intentionally gone (see spec). `git` is added because the workspace is a git checkout and the agent pulls it.

- [ ] **Step 3: Update README tool table**

In `README.md`, remove the `claude` row from the Added Tools table and replace the two `clawdbot/clawdbot` links with `https://github.com/openclaw/openclaw`. Change the Versioning example to `2026.9.4-k8s`.

- [ ] **Step 4: Build locally to catch apt or install failures**

```bash
cd ~/git/clawdbot-k8s
docker build --platform linux/amd64 -t clawdbot-k8s:test .
docker run --rm --platform linux/amd64 --entrypoint sh clawdbot-k8s:test -c \
  'id -u; node --version; ls /app/dist/index.js; kubectl version --client --output=yaml | head -3; flux version --client; talosctl version --client; gh --version; doppler --version; yq --version; git --version'
```

Expected: `1000`, Node `v24.16` or newer, the file exists, kubectl `1.34.x`, every tool prints a version. If `id -u` prints `0`, `USER node` is missing.

- [ ] **Step 5: Push the branch and open a PR**

```bash
git add Dockerfile README.md
git commit -m "feat(docker): rebuild on openclaw 2026.9.4, drop claude, run as node"
git push -u origin feat/openclaw-2026.9
gh pr create --title "feat(docker): rebuild on OpenClaw 2026.9.4" --body "Supersedes #29. Drops claude CLI, restores USER node, kubectl 1.34. Workspace edits follow in a separate PR."
```

Then close the superseded Renovate PR:

```bash
gh pr close 29 --comment "Superseded by the 2026.9.4 rebuild PR."
```

- [ ] **Step 6: Merge and verify the published tag**

After Craighton approves and merges (Dockerfile changes need his review per the workspace's own rule in TOOLS.md):

```bash
gh run watch --exit-status
curl -s "https://hub.docker.com/v2/repositories/craighton/clawdbot-k8s/tags?page_size=5" | jq -r '.results[].name'
```

Expected: `2026.9.4-k8s` in the list.

---

### Task 2: Workspace Markdown for the 2026.9 runtime

**Files:**
- Modify: `~/git/clawdbot-k8s/AGENTS.md`
- Delete: `~/git/clawdbot-k8s/TOOLS.md`, `HEARTBEAT.md`, `scripts/check-claude-usage.sh`, `memory/claude-usage.md`

**Interfaces:**
- Consumes: nothing from Task 1; independent branch.
- Produces: a workspace `openclaw doctor` has nothing to migrate in, and no instructions that reference Anthropic usage windows or `claude`.

- [ ] **Step 1: Branch**

```bash
cd ~/git/clawdbot-k8s && git checkout main && git pull && git checkout -b docs/openclaw-2026.9-workspace
```

- [ ] **Step 2: Rewrite AGENTS.md**

Replace the `## Usage Management` section (from that heading to the end of the "Pause & Resume Flow" list) with:

```markdown
## Model and Usage

The gateway runs on a ChatGPT subscription via the `openai` provider. There is no
5-hour usage block to watch. If a request fails with a rate-limit error, wait and
retry once, then tell Craighton instead of looping.
```

Append the TOOLS.md content as a new section. Copy everything in `TOOLS.md` below its
first paragraph (starting at `## ⚠️ CRITICAL: Git Workflow for Infrastructure Repos`)
verbatim under a new heading:

```markdown
## Tools

<pasted TOOLS.md body>
```

Then edit inside the pasted text: replace `Clawdbot` with `OpenClaw` in prose, and
replace the sentence `\`clawdbot-k8s\` - **Dockerfile only** (triggers Docker image builds)` with
`` `clawdbot-k8s` - Dockerfile changes trigger image builds; Markdown changes do not ``.

Append the HEARTBEAT.md startup check as another section:

```markdown
## Startup Check

If `memory/pending-restart.md` exists at the start of a session or heartbeat:
1. Read it to restore context from before the restart.
2. Run any verification steps listed in it.
3. Message Craighton on Discord with the result.
4. Delete the file after notifying.

If verification fails: ping Craighton, try to fix it yourself, increment `retryCount`
in the file, and ask for manual intervention once `retryCount >= 3`.
```

- [ ] **Step 3: Delete retired files**

```bash
git rm TOOLS.md HEARTBEAT.md scripts/check-claude-usage.sh memory/claude-usage.md
```

- [ ] **Step 4: Check for leftover references**

```bash
grep -rniE "claude|anthropic|5h block|models status|TOOLS\.md|HEARTBEAT\.md" --include="*.md" --include="*.sh" . | grep -v "^./MIGRATION.md" | grep -v "^./memory/claude-code.md"
```

Expected: no hits outside `memory/claude-code.md` (historical note, leave it) and `MIGRATION.md` (historical). Fix any others by removing the sentence.

- [ ] **Step 5: PR and merge**

```bash
git add -A
git commit -m "docs(workspace): fold TOOLS and HEARTBEAT into AGENTS, drop Anthropic usage rules"
git push -u origin docs/openclaw-2026.9-workspace
gh pr create --fill
```

Markdown-only, so this PR can merge without waiting on Task 1.

---

### Task 3: Remove the dead SOPS secret

**Files:**
- Delete: `kubernetes/apps/agents/clawdbot/app/secret.sops.yaml`

**Interfaces:**
- Produces: nothing; the file has been commented out of `kustomization.yaml` since the Doppler migration and only holds a stale Anthropic key ciphertext.

- [ ] **Step 1: Confirm it is unreferenced, then remove it**

```bash
cd ~/git/home-cluster/.worktrees/openclaw-revival
grep -rn "secret.sops.yaml" kubernetes/apps/agents/clawdbot/app/kustomization.yaml
git rm kubernetes/apps/agents/clawdbot/app/secret.sops.yaml
```

Expected: the grep shows only the commented-out line. Then edit `kustomization.yaml` to drop the two comment lines so it reads:

```yaml
---
# yaml-language-server: $schema=https://json.schemastore.org/kustomization
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./helmrelease.yaml
  - ./externalsecret.yaml
```

- [ ] **Step 2: Build check**

```bash
kustomize build kubernetes/apps/agents/clawdbot/app | yq '.kind' | sort | uniq -c
```

Expected: one `ExternalSecret`, one `HelmRelease`, no `Secret`.

---

### Task 4: HelmRelease rewrite

**Files:**
- Modify: `kubernetes/apps/agents/clawdbot/app/helmrelease.yaml`

**Interfaces:**
- Consumes: Secret `clawdbot-secret` (existing ExternalSecret, provides `OPENCLAW_GATEWAY_TOKEN` and `DISCORD_BOT_TOKEN`), image tag `2026.9.4-k8s` (Task 1, also used by the init container for `jq`), existing PVC `clawdbot-workspace` holding the old `.openclaw/openclaw.json`.
- Produces: Deployment `clawdbot` with one replica, PVC `clawdbot-home` (10Gi, `synology-iscsi`).

- [ ] **Step 1: Replace `spec.values` and add drift detection**

Full file:

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/helm.toolkit.fluxcd.io/helmrelease_v2.json
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: clawdbot
  namespace: agents
spec:
  interval: 30m
  chart:
    spec:
      chart: app-template
      version: 3.7.3
      sourceRef:
        kind: HelmRepository
        name: bjw-s
        namespace: flux-system
  install:
    remediation:
      retries: 3
  upgrade:
    cleanupOnFail: true
    remediation:
      retries: 3
  # The deployment was hand-scaled to 0 while shut down. Helm's three-way merge
  # leaves an unchanged replicas field alone, so drift detection is what puts it back to 1.
  driftDetection:
    mode: enabled
  values:
    controllers:
      clawdbot:
        replicas: 1
        strategy: Recreate
        annotations:
          reloader.stakater.com/auto: "true"
        pod:
          securityContext:
            runAsUser: 1000
            runAsGroup: 1000
            fsGroup: 1000
            fsGroupChangePolicy: OnRootMismatch
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: kubernetes.io/arch
                        operator: In
                        values:
                          - amd64
        initContainers:
          seed-home:
            # Same image as the app so jq is available. Runs as root to chown.
            image:
              repository: craighton/clawdbot-k8s
              tag: 2026.9.4-k8s
            securityContext:
              runAsUser: 0
            command:
              - sh
              - -c
              - |
                set -eu
                # First boot only: carry the old config and dotfiles from the NFS home
                # onto the new block-storage home. Marker file makes this idempotent.
                if [ ! -f /home/node/.seeded ]; then
                  mkdir -p /home/node/.openclaw
                  # Strip the Anthropic auth profile, the sonnet model pin and allowlist,
                  # and contextPruning (unconfirmed in the 2026.9 strict schema).
                  jq 'del(.auth, .agents.defaults.model, .agents.defaults.models, .agents.defaults.contextPruning)' \
                    /oldhome/.openclaw/openclaw.json > /home/node/.openclaw/openclaw.json
                  [ -d /oldhome/.openclaw/credentials ] && cp -a /oldhome/.openclaw/credentials /home/node/.openclaw/
                  for d in .talos .doppler .config/sops; do
                    if [ -d "/oldhome/$d" ]; then mkdir -p "/home/node/$(dirname $d)"; cp -a "/oldhome/$d" "/home/node/$d"; fi
                  done
                  [ -f /oldhome/.gitconfig ] && cp -a /oldhome/.gitconfig /home/node/.gitconfig
                  # The workspace checkout on NFS was written as root; the gateway now runs as 1000.
                  chown -R 1000:1000 /oldhome/clawd
                  date -u +%FT%TZ > /home/node/.seeded
                fi
                chown -R 1000:1000 /home/node
        containers:
          clawdbot:
            image:
              repository: craighton/clawdbot-k8s
              tag: 2026.9.4-k8s
            env:
              HOME: /home/node
              TERM: xterm-256color
              TZ: America/Denver
              OPENCLAW_STATE_DIR: /home/node/.openclaw
              OPENCLAW_GATEWAY_PORT: "18789"
            envFrom:
              - secretRef:
                  name: clawdbot-secret
            command:
              - node
              - dist/index.js
              - gateway
              - --bind
              - lan
              - --port
              - "18789"
            probes:
              startup:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /startupz
                    port: &port 18789
                  periodSeconds: 10
                  failureThreshold: 30
              readiness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /startupz
                    port: *port
                  periodSeconds: 10
                  timeoutSeconds: 5
                  failureThreshold: 3
              liveness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /healthz
                    port: *port
                  periodSeconds: 30
                  timeoutSeconds: 10
                  failureThreshold: 3
            resources:
              requests:
                cpu: 100m
                memory: 512Mi
              limits:
                memory: 2Gi

    service:
      clawdbot:
        controller: clawdbot
        ports:
          http:
            port: *port

    ingress:
      clawdbot:
        enabled: true
        className: internal
        annotations:
          cert-manager.io/cluster-issuer: letsencrypt-production
        hosts:
          - host: clawd.${SECRET_DOMAIN}
            paths:
              - path: /
                service:
                  identifier: clawdbot
                  port: http
        tls:
          - hosts:
              - clawd.${SECRET_DOMAIN}
            secretName: clawd-tls

    persistence:
      # Whole home on block storage: OpenClaw's SQLite state and hard-link locks
      # are not safe on NFS (openclaw/openclaw#81089).
      home:
        enabled: true
        type: persistentVolumeClaim
        storageClass: synology-iscsi
        accessMode: ReadWriteOnce
        size: 10Gi
        retain: true
        advancedMounts:
          clawdbot:
            seed-home:
              - path: /home/node
            clawdbot:
              - path: /home/node
      # Existing NFS volume. Kept intact as rollback; only clawd/ is exposed to
      # the gateway, which hides the stale .openclaw state next to it.
      oldhome:
        enabled: true
        type: persistentVolumeClaim
        existingClaim: clawdbot-workspace
        advancedMounts:
          clawdbot:
            seed-home:
              - path: /oldhome
            clawdbot:
              - path: /home/node/clawd
                subPath: clawd
```

- [ ] **Step 2: Render with the real chart and inspect**

```bash
cd ~/git/home-cluster/.worktrees/openclaw-revival
yq '.spec.values' kubernetes/apps/agents/clawdbot/app/helmrelease.yaml > "$SCRATCH/values.yaml"
helm template clawdbot oci://ghcr.io/bjw-s/helm/app-template --version 3.7.3 -n agents -f "$SCRATCH/values.yaml" > "$SCRATCH/rendered.yaml"
yq 'select(.kind=="Deployment") | .spec.template.spec | {strategy: .strategy, sc: .securityContext, init: [.initContainers[].name], mounts: [.containers[0].volumeMounts[] | .mountPath + " <- " + .name + " " + (.subPath // "")]}' "$SCRATCH/rendered.yaml"
yq 'select(.kind=="Deployment") | .spec | {replicas, strategy: .strategy.type}' "$SCRATCH/rendered.yaml"
yq 'select(.kind=="PersistentVolumeClaim") | .metadata.name + " " + .spec.storageClassName' "$SCRATCH/rendered.yaml"
```

Expected: init container `seed-home`; main container mounts `/home/node <- home` and `/home/node/clawd <- oldhome clawd`; PVC `clawdbot-home synology-iscsi`; replicas 1; strategy `Recreate`; pod securityContext with fsGroup 1000. If `subPath` is missing from the mount, app-template 3.7.3 needs `subPath` spelled exactly as above under `advancedMounts`; check the chart's `values.yaml` with `helm show values oci://ghcr.io/bjw-s/helm/app-template --version 3.7.3 | grep -n subPath`.

- [ ] **Step 3: Confirm no legacy env names**

```bash
grep -n "CLAWDBOT_" kubernetes/apps/agents/clawdbot/app/helmrelease.yaml || echo clean
```

Expected: `clean`.

- [ ] **Step 4: Schema validation like CI**

```bash
brew list kubeconform >/dev/null 2>&1 || brew install kubeconform
kustomize build kubernetes/apps/agents/clawdbot/app --load-restrictor=LoadRestrictionsNone \
  | kubeconform -strict -ignore-missing-schemas -skip Secret \
    -schema-location default \
    -schema-location 'https://kubernetes-schemas.pages.dev/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' -verbose
```

Expected: every resource `PASS`.

- [ ] **Step 5: Stage and show the diff**

```bash
git add kubernetes/apps/agents/clawdbot docs/superpowers
git status --short
git diff --cached --stat
```

Stop here and hand the diff to Craighton. Commit and PR happen on his word.

---

### Task 5: Deploy and first boot

**Files:** none. Operational.

**Interfaces:**
- Consumes: merged PRs from Tasks 1 to 4; `OPENCLAW_GATEWAY_TOKEN` already in Doppler (done 2026-09-11).

- [ ] **Step 1: Confirm the gateway token has synced**

```bash
k() { kubectl --context hancockfam-local --insecure-skip-tls-verify "$@"; }
k -n agents get secret clawdbot-secret -o json | jq -r '.data | keys[]' | grep OPENCLAW_GATEWAY_TOKEN
```

Expected: the key prints. If not, `k -n agents annotate externalsecret clawdbot-secret force-sync=$(date +%s) --overwrite` and re-check.

- [ ] **Step 2: Merge the home-cluster PR and reconcile**

```bash
flux --context hancockfam-local reconcile source git home-kubernetes -n flux-system
flux --context hancockfam-local reconcile kustomization clawdbot -n flux-system --with-source
k -n agents get pvc,pods -w
```

Expected within ~3 minutes: PVC `clawdbot-home` Bound, pod `clawdbot-*` passes `Init:0/1`, then `Running 1/1`. If the pod sticks in `Init`, `k -n agents logs <pod> -c seed-home`.

- [ ] **Step 3: Check the seed landed**

```bash
POD=$(k -n agents get pod -l app.kubernetes.io/name=clawdbot -o name | head -1)
k -n agents exec $POD -- sh -c 'id -u; cat /home/node/.seeded; ls -la /home/node; ls /home/node/.openclaw; head -3 /home/node/clawd/SOUL.md'
```

Expected: `1000`, a timestamp, `.talos .doppler .config .gitconfig .openclaw clawd` present, `openclaw.json` and `credentials/` present, SOUL.md prints. Then confirm the strip worked:

```bash
k -n agents exec $POD -- jq '{auth, model: .agents.defaults.model, models: .agents.defaults.models, discord: .channels.discord.guilds | keys}' /home/node/.openclaw/openclaw.json
```

Expected: `auth`, `model`, `models` all `null`; the Discord guild key present.

- [ ] **Step 4: Gateway health**

```bash
k -n agents exec $POD -- sh -c 'curl -fsS localhost:18789/healthz; echo; curl -fsS localhost:18789/startupz; echo'
k -n agents logs $POD --tail=50 | grep -iE "error|warn|OPENCLAW_LEGACY|discord"
```

Expected: both endpoints return 200 JSON; no `OPENCLAW_LEGACY_ENV_VARS` warning; Discord reports connected or waiting on a model (model auth is next).

- [ ] **Step 5: ChatGPT login (needs Craighton at a browser)**

```bash
k -n agents exec -it $POD -- node dist/index.js models auth login --provider openai --device-code
```

Read the code and URL aloud in chat. Craighton opens the URL, signs in to ChatGPT, enters the code. Expected: the command exits with a saved-profile message.

- [ ] **Step 6: Pick a model the account actually has**

```bash
k -n agents exec $POD -- node dist/index.js models list --provider openai
```

If `openai/gpt-5.6-sol` is not listed, set the best listed non-1M-context model:

```bash
k -n agents exec $POD -- node dist/index.js models set openai/<id-from-list>
```

- [ ] **Step 7: Doctor pass**

```bash
k -n agents exec $POD -- node dist/index.js doctor --fix --non-interactive
```

Expected: no legacy session import (fresh state); reports config OK. Anything it rewrites in `openclaw.json` is fine; the seed is first-boot only.

- [ ] **Step 7a: Port the HEARTBEAT.md prose to the heartbeat prompt**

HEARTBEAT.md is no longer read; its startup-check prose maps to `agents.defaults.heartbeat.prompt`. Set it inside the pod (config, not git):

```bash
k -n agents exec $POD -- node dist/index.js config set agents.defaults.heartbeat.prompt \
  "If memory/pending-restart.md exists in the workspace, this counts as needing attention: read it, run its verification steps, message Craighton on Discord with the result, then delete it. Otherwise reply HEARTBEAT_OK."
k -n agents exec $POD -- jq '.agents.defaults.heartbeat' /home/node/.openclaw/openclaw.json
```

Expected: `every: "3h"` still present and the new `prompt`. If `config set` is not a valid subcommand in 2026.9, edit the file with `jq '.agents.defaults.heartbeat.prompt = "..."'` and restart the pod.

- [ ] **Step 7b: Verify the startup-wake CLI still works**

The old `scripts/startup-wake.sh` sent an explicit startup message after boot. Test the command it relies on:

```bash
k -n agents exec $POD -- node dist/index.js agent --agent main --deliver --channel discord \
  --message "Startup check test - reply with one line confirming you received this."
```

Expected: a Discord message from the bot. If the subcommand or flags have changed, look up the current form with `node dist/index.js agent --help` and note the working invocation. Follow-up (separate PRs, after this plan): update `scripts/startup-wake.sh` to poll `/healthz` and use the working invocation, then restore it as a background step in the container command.

- [ ] **Step 8: Pull the workspace to the merged Markdown**

```bash
k -n agents exec $POD -- git -C /home/node/clawd pull --ff-only
k -n agents exec $POD -- ls /home/node/clawd
```

Expected: fast-forward from `90044c5`; `TOOLS.md` and `HEARTBEAT.md` gone. If `pull` complains about `dubious ownership`, the chown in the init container did not cover `.git`; run `k -n agents exec $POD -- git config --global --add safe.directory /home/node/clawd` and retry.

- [ ] **Step 9: Functional checks**

1. Craighton mentions the bot in the allowed Discord guild. Expected: 👀 reaction, then a reply.
2. `k -n agents exec $POD -- kubectl get pods -A | head -3` works through the existing `clawdbot-cluster-role`.
3. `k -n agents exec $POD -- talosctl --nodes <cp-ip> version | head -3` works, proving `.talos/config` seeded.
4. Within 3 hours, the log shows a heartbeat run: `k -n agents logs $POD | grep -i heartbeat`.

- [ ] **Step 10: Record outcome**

Update the memory file `openclaw-revival-state.md` in the Claude memory directory with the deployed version, the model chosen, and the date. Then post a one-line summary in chat.

---

### Task 6: Rollback procedure (only if Task 5 fails and cannot be fixed in place)

- [ ] **Step 1: Revert the home-cluster PR**

```bash
gh pr list --state merged --search "openclaw revival" --limit 1
git revert <merge-sha> && git push
flux --context hancockfam-local reconcile kustomization clawdbot -n flux-system --with-source
```

- [ ] **Step 2: Scale back to zero**

The reverted manifest still has no `replicas: 0`, so the old image would start. Scale it down:

```bash
k -n agents scale deploy clawdbot --replicas=0
```

- [ ] **Step 3: Leave the new PVC**

`clawdbot-home` has `retain: true`; delete it by hand only if the revival is abandoned for good.

---

## Self-Review

**Spec coverage.** Storage table → Task 4 persistence block. Configuration carry-over → init container in Task 4. Secrets → Global Constraints and Task 5 Step 1. Image changes → Task 1. Deployment changes (probes, strategy, security context, replicas, env, command) → Task 4. Workspace adjustments → Task 2, plus `memory/.ha-db-password` deletion: **gap**, added below as Task 5 Step 8a. First-boot runbook → Task 5. Rollback → Task 6. Renovate GitHub Actions PRs 25 to 28 and 30 → **not in a task**; they are unrelated to the revival, leave for Craighton to merge from the GitHub UI, noted here so nobody hunts for them.

- [ ] **Task 5 Step 8a: Remove the plaintext password file from the NAS**

```bash
k -n agents exec $POD -- sh -c 'rm -f /home/node/clawd/memory/.ha-db-password && echo removed'
```

**Placeholder scan.** `<id-from-list>`, `<pod>`, `<cp-ip>`, `<merge-sha>` are runtime values, filled from the preceding command's output. No TBDs.

**Name consistency.** PVC `clawdbot-home` is what app-template names `persistence.home` for release `clawdbot`; Task 5 Step 2 expects that name. Init container `seed-home` used consistently in `advancedMounts`. Marker file `/home/node/.seeded` in both the init script and Task 5 Step 3.
