# Kubernetes Cluster Upgrade Automation

![License](https://img.shields.io/badge/license-MIT-blue)
![Kubernetes](https://img.shields.io/badge/kubernetes-v1.35-326CE5?logo=kubernetes&logoColor=white)
![Ansible](https://img.shields.io/badge/ansible-automation-EE0000?logo=ansible&logoColor=white)
![Claude Code](https://img.shields.io/badge/AI-Claude_Code-8A2BE2)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)

AI-powered Kubernetes cluster upgrades for bare metal homelabs.

GitHub Actions pipelines detect new versions weekly, auto-upgrade patches, and orchestrate minor version upgrades with **Claude Code AI analysis and approval workflows** — all backed by Ansible playbooks.

This is running on my personal homelab: an 8-node bare metal cluster (3 control plane + 5 workers) running Cilium, Rook-Ceph, ArgoCD, and Vault.

## Stack

| Layer | Tool | Role |
|-------|------|------|
| Orchestration | GitHub Actions | Scheduling, triggers, approvals |
| AI | Claude Code CLI | Changelog analysis, pre/post-upgrade fixes, diagnosis |
| Execution | Ansible | Sequential node-by-node upgrade |
| Notifications | Discord | Per-node progress, reports, approval links |

## Architecture

```
GitHub Actions (Saturday 03:00 UTC)
    │
    ├─ version-check.yaml
    │   ├─ New patch >7 days old?  → Auto-trigger upgrade-cluster.yaml
    │   ├─ New patch <7 days old?  → Discord: "Available, waiting N days"
    │   ├─ New minor, patch ≥ .2?  → Claude Code session → Discord report + approval link
    │   └─ No updates              → Silent
    │
    └─ upgrade-cluster.yaml (triggered by version-check or manual)
        ├─ Pre-flight: node health, etcd quorum, pod checks
        ├─ Pre-upgrade fixes: Claude Code session executes checklist (minor only)
        ├─ Upgrade: Ansible playbook (sequential CP → workers)
        ├─ Post-upgrade: verify nodes, pods, ArgoCD, Vault unseal
        └─ Discord: full report with run link
```

## Upgrade History

This system has been running in production on my homelab:

| Date | From | To | Notes |
|------|------|----|-------|
| 2026-04-05 | v1.31.14 | v1.32.13 | First automated upgrade |
| 2026-04-05 | v1.32.13 | v1.33.10 | |
| 2026-04-05 | v1.33.10 | v1.34.6 | |
| 2026-04-05 | v1.34.6 | v1.35.3 | Cilium v1.16.5 → v1.17.14 pre-upgrade fix by Claude Code |

Four consecutive minor version upgrades in a single day, fully automated.

## How AI is Used

There is no direct Anthropic API integration in this pipeline. Instead, GitHub Actions shells out to the **Claude Code CLI** (`claude`), which spawns a full agentic Claude Code session on the self-hosted runner. Claude Code handles everything within that session autonomously — running `kubectl`, editing files, making web requests, committing to Git — before returning control to the workflow.

Claude Code sessions run at three points in the pipeline:

**1. Minor version analysis (`version-check.yaml`)**
When a new minor version is detected, a Claude Code session:
- Fetches and reads the full Kubernetes changelog
- Scans the cluster for deprecated API usage via `kubectl`
- Checks Cilium, Rook-Ceph, and Vault compatibility matrices
- Writes a pre-upgrade checklist to `/tmp/k8s-upgrade-checklist-vX.Y.Z.txt`
- Posts a detailed report to Discord with an approval link

**2. Pre-upgrade fixes (`upgrade-cluster.yaml`, minor only)**
After approval, a Claude Code session executes the checklist:
- Upgrades CNI if incompatible with the target version
- Fixes deprecated APIs in manifests
- Commits and pushes changes, then waits for ArgoCD to sync

**3. Post-upgrade diagnosis (`upgrade-cluster.yaml`)**
After Ansible completes, a Claude Code session:
- Verifies all nodes are on the target version
- Checks for unhealthy pods and diagnoses issues
- Confirms ArgoCD apps are Synced and Healthy
- Posts a final report to Discord

The invocation pattern in each workflow step looks like this:
```bash
claude --dangerously-skip-permissions -p "
  Your task description here...
  Post results to Discord webhook $DISCORD_UPGRADE_WEBHOOK
" 2>&1 || true
```

> **Note on `--dangerously-skip-permissions`:** This flag allows Claude Code to run non-interactively without confirmation prompts for each action. It is required in a CI environment where there is no human at the keyboard. Claude Code runs inside a self-hosted runner with network and filesystem access scoped to the cluster — review what access your runner has before enabling this.

## Quick Start

### 1. Prerequisites

On your self-hosted GitHub Actions runner:
```bash
kubectl   # with kubeconfig for your cluster
ansible   # for running playbooks
claude    # Claude Code CLI — https://claude.ai/code
gh        # GitHub CLI (authenticated to this repo)
ssh       # key-based access to all cluster nodes
```

Claude Code must be authenticated on your runner. Log in once interactively:
```bash
claude login
```

This stores credentials locally on the runner. The `claude` CLI will use them for all subsequent non-interactive sessions triggered by GitHub Actions.

### 2. Clone and configure

```bash
git clone https://github.com/joysontech/kubernetes-upgrade-automation
cd kubernetes-upgrade-automation

cp inventory.ini.example inventory.ini
cp group_vars/all.yml.example group_vars/all.yml
```

Edit `inventory.ini` with your node IPs and SSH details.
Edit `group_vars/all.yml` with your target versions and Discord webhook.

### 3. GitHub Actions secrets

Set in repo → Settings → Secrets and variables → Actions:

| Secret | Description |
|--------|-------------|
| `DISCORD_UPGRADE_WEBHOOK` | Discord webhook URL for upgrade notifications |
| `BECOME_PASS` | sudo password for Ansible become |

> **Discord webhooks:** Create one in Discord → channel settings → Integrations → Webhooks.
> The Ansible playbook also reads `discord_webhook` from `group_vars/all.yml` — set this to the same URL.

### 4. Set your paths

In `.github/workflows/upgrade-cluster.yaml`, set `K8S_UPGRADE_DIR` to where this repo is cloned on your runner.

In `.github/workflows/version-check.yaml`, set `RUNNER_USER` and `RUNNER_IP` to your runner host.

### 5. Set up the approval server (minor upgrades only)

Minor version upgrades require a one-click approval. The Claude Code session posts a Discord message with a link to `http://YOUR_RUNNER_IP:9095/approve?version=vX.Y.Z`. Clicking it creates `/tmp/k8s-upgrade-approved` on your runner host, which the next Sunday version check picks up and uses to trigger the upgrade.

A minimal approval server (Python):
```python
# approval_server.py — run on your runner host
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        params = parse_qs(urlparse(self.path).query)
        version = params.get("version", ["unknown"])[0]
        with open("/tmp/k8s-upgrade-approved", "w") as f:
            f.write(version)
        self.send_response(200)
        self.end_headers()
        self.wfile.write(f"Approved: {version}".encode())

HTTPServer(("0.0.0.0", 9095), Handler).serve_forever()
```

Run it as a systemd service or in a tmux session on your runner host.

### 6. Trigger

```bash
# Check for updates manually
gh workflow run version-check.yaml

# Trigger a specific upgrade manually
gh workflow run upgrade-cluster.yaml \
  -f target_version=v1.35.4 \
  -f apt_version=1.35.4-1.1 \
  -f upgrade_type=patch \
  -f run_pre_fixes=false
```

## Upgrade Sequence

```
Phase 1: Pre-Upgrade
  ✓ Verify all nodes Ready
  ✓ Verify etcd quorum healthy
  ✓ Snapshot etcd to /var/backups/etcd
  ✓ Backup PKI certificates
  → Discord: "K8s Upgrade Starting"

Phase 2: Control Plane (sequential, ~10 min each)
  First CP:       kubeadm upgrade apply vX.Y.Z → drain → kubelet upgrade → uncordon → verify
  Remaining CPs:  kubeadm upgrade node          → drain → kubelet upgrade → uncordon → verify
  → Discord: "Control Plane Upgraded — M01 / M02 / M03"

Phase 3: Workers (sequential, ~5 min each)
  Each worker: kubeadm upgrade node → drain → kubelet upgrade → uncordon → verify
  → Discord: "Worker Upgraded — W01 / W02 / ..."

Phase 4: Post-Upgrade
  ✓ All nodes on target version
  ✓ All pods Running/Completed
  ✓ ArgoCD apps Synced
  ✓ Vault auto-unseal if needed
  ✓ Smoke test (create/delete nginx pod)
  → Discord: "K8s Upgrade Complete"
```

## Minor Version Flow

```
1. Version check detects vX.Y+1.2 (patch ≥ .2 — community has had time to find bugs)
   Claude Code session: fetch changelog → scan deprecated APIs → check CNI/storage/Vault compat
   → Writes checklist to /tmp/k8s-upgrade-checklist-vX.Y+1.2.txt
   → Discord: detailed report + approval link

2. You click the approval link
   → Approval marker created at /tmp/k8s-upgrade-approved on runner host

3. Next Sunday version check picks up the marker
   → Triggers upgrade-cluster.yaml with run_pre_fixes=true
   → Claude Code session: executes checklist (fix deprecations, upgrade CNI if needed, commit + push)
   → Waits for ArgoCD to sync

4. Ansible runs sequential upgrade (CP → workers)

5. Claude Code session: post-upgrade verification
   → Discord: "Upgrade complete — all nodes on vX.Y+1.2"
```

> **Why wait for patch ≥ .2?** Kubernetes .0 and .1 releases often surface edge-case bugs quickly. Waiting for .2 means the community has already found and fixed the most common issues before your cluster upgrades.

## Important Constraints

- **Never skip minor versions** — upgrade one at a time (1.34 → 1.35 → 1.36, never 1.34 → 1.36)
- **kube-vip** — pin to a known-good version; static pod upgrades require manual intervention
- **Cilium** — always check the [compatibility matrix](https://docs.cilium.io/en/stable/network/kubernetes/compatibility/) before a minor upgrade
- **Rook-Ceph** — ensure 2+ OSD nodes are healthy at all times; drain pauses until volumes detach
- **Vault** — may need auto-unseal after worker drains cause pod restarts; handled in post-upgrade job
- **etcd quorum** — never upgrade more than 1 control plane node at a time
- **Drain flags** — `--disable-eviction` bypasses PDBs (e.g. Kyverno); `--timeout=300s` allows time for Rook-Ceph volume detach

## File Structure

```
kubernetes-upgrade-automation/
  .github/workflows/
    version-check.yaml        # Weekly version detection + auto-trigger
    upgrade-cluster.yaml      # Full upgrade pipeline (4 jobs)
  ansible.cfg                 # forks=1, serial=1 (never parallel)
  inventory.ini.example       # Node inventory template → copy to inventory.ini
  group_vars/
    all.yml.example           # Variables template → copy to all.yml
  upgrade-cluster.yaml        # Ansible upgrade playbook
  rollback.yaml               # Ansible emergency rollback playbook
  roles/
    pre-upgrade/              # Health checks, etcd snapshot, PKI backup
    upgrade-control-plane/    # CP upgrade (kubeadm apply vs node)
    upgrade-workers/          # Worker drain + upgrade + uncordon
    post-upgrade/             # Verification + smoke test
    rollback/                 # etcd restore + version downgrade
```

## Rollback

If an upgrade goes wrong, restore from the etcd snapshot taken in Phase 1:

```bash
ansible-playbook rollback.yaml \
  -e ansible_become_pass=YOUR_SUDO_PASS \
  -e k8s_rollback_version=1.34.6-1.1
```

## License

MIT — adapt freely for your own cluster.
