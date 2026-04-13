# Kubernetes Cluster Upgrade Automation

AI-powered Kubernetes cluster upgrades for bare metal homelabs.

GitHub Actions pipelines detect new versions weekly, auto-upgrade patches, and orchestrate minor version upgrades with **Claude Code AI analysis and approval workflows** — all backed by Ansible playbooks.

## Stack

| Layer | Tool | Role |
|-------|------|------|
| Orchestration | GitHub Actions | Scheduling, triggers, approvals |
| AI | Claude Code | Changelog analysis, pre/post-upgrade fixes, diagnosis |
| Execution | Ansible | Sequential node-by-node upgrade |
| Notifications | Discord | Per-node progress, reports, approval links |

## Architecture

```
GitHub Actions (Saturday 03:00 UTC)
    │
    ├─ version-check.yaml
    │   ├─ New patch >7 days old?  → Auto-trigger upgrade-cluster.yaml
    │   ├─ New patch <7 days old?  → Discord: "Available, waiting N days"
    │   ├─ New minor, patch ≥ .2?  → Claude Code analysis → Discord report + approval link
    │   └─ No updates              → Silent
    │
    └─ upgrade-cluster.yaml (triggered by version-check or manual)
        ├─ Pre-flight: node health, etcd quorum, pod checks
        ├─ Pre-upgrade fixes: Claude Code executes checklist (minor only)
        ├─ Upgrade: Ansible playbook (sequential CP → workers)
        ├─ Post-upgrade: verify nodes, pods, ArgoCD, Vault unseal
        └─ Discord: full report with run link
```

## Cluster Compatibility

Designed for **kubeadm-managed bare metal clusters**. Tested on:

- 8-node cluster (3 control plane + 5 workers)
- Cilium CNI (kube-proxy replacement, VXLAN mode)
- Rook-Ceph storage
- kube-vip (HA control plane VIP)
- ArgoCD (GitOps, App-of-Apps)
- HashiCorp Vault (auto-unseal after drains)

Adapt `inventory.ini` and `group_vars/all.yml` for your own setup.

## Quick Start

### 1. Prerequisites

On your self-hosted GitHub Actions runner:
```bash
kubectl   # with kubeconfig for your cluster
ansible   # for running playbooks
claude    # Anthropic Claude Code CLI (https://claude.ai/code)
gh        # GitHub CLI (authenticated)
ssh       # key-based access to all cluster nodes
```

### 2. Clone and configure

```bash
git clone https://github.com/joyson-fernandes/kubernetes-upgrade-automation
cd kubernetes-upgrade-automation

cp inventory.ini.example inventory.ini
cp group_vars/all.yml.example group_vars/all.yml
```

Edit `inventory.ini` with your node IPs and SSH details.
Edit `group_vars/all.yml` with your target versions and Discord webhook.

### 3. GitHub Actions secrets

Set in repo → Settings → Secrets:

| Secret | Description |
|--------|-------------|
| `DISCORD_WEBHOOK` | Discord webhook URL for general alerts |
| `DISCORD_UPGRADE_WEBHOOK` | Discord webhook URL for upgrade channel |
| `BECOME_PASS` | sudo password for Ansible become |

### 4. Set your runner path

In `.github/workflows/upgrade-cluster.yaml` set `K8S_UPGRADE_DIR` to where you cloned this repo on your runner.

In `.github/workflows/version-check.yaml` set `RUNNER_IP` and `RUNNER_USER`.

### 5. Trigger

```bash
# Check for updates
gh workflow run version-check.yaml

# Manual upgrade
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
  First CP: kubeadm upgrade apply vX.Y.Z → drain → kubelet upgrade → uncordon → verify
  Remaining CPs: kubeadm upgrade node → drain → kubelet upgrade → uncordon → verify
  → Discord: "Control Plane Upgraded — M01/M02/M03"

Phase 3: Workers (sequential, ~5 min each)
  Each worker: kubeadm upgrade node → drain → kubelet upgrade → uncordon → verify
  → Discord: "Worker Upgraded — W01/W02/..."

Phase 4: Post-Upgrade
  ✓ All nodes on target version
  ✓ All pods Running/Completed
  ✓ ArgoCD apps Synced
  ✓ Vault auto-unseal if needed
  ✓ Smoke test
  → Discord: "K8s Upgrade Complete"
```

## Minor Version Flow

```
1. Claude Code: fetch changelog → scan deprecated APIs → check CNI/storage/Vault compat
   → Discord report + approval link

2. You click approve → approval marker created on runner

3. Next weekly check picks up approval
   → Claude Code: fix deprecations, upgrade CNI, commit + push, wait for ArgoCD
   → Ansible: sequential upgrade

4. Claude Code: post-upgrade verification + Discord report
```

## Important Constraints

- **Never skip minor versions** — upgrade one at a time (1.34 → 1.35, not 1.34 → 1.36)
- **kube-vip** — pin to a known-good version; static pods don't auto-upgrade
- **Cilium** — always check the compatibility matrix before a minor upgrade
- **Rook-Ceph** — ensure 2+ OSD nodes healthy during drain
- **Vault** — may need auto-unseal after worker drains restart vault pods
- **etcd quorum** — never upgrade more than 1 control plane node at a time
- **Drain flags** — `--disable-eviction` bypasses PDBs; `--timeout=300s` for storage volume detach

## File Structure

```
kubernetes-upgrade-automation/
  .github/workflows/
    version-check.yaml        # Weekly version detection + auto-trigger
    upgrade-cluster.yaml      # Full upgrade pipeline
  ansible.cfg                 # forks=1, serial=1
  inventory.ini.example       # Node inventory template → copy to inventory.ini
  group_vars/
    all.yml.example           # Variables template → copy to all.yml
  upgrade-cluster.yaml        # Ansible upgrade playbook
  rollback.yaml               # Ansible emergency rollback
  roles/
    pre-upgrade/              # Health checks, etcd + PKI backup
    upgrade-control-plane/    # CP upgrade (apply vs node)
    upgrade-workers/          # Worker upgrade (drain + upgrade)
    post-upgrade/             # Verification + smoke test
    rollback/                 # etcd restore + downgrade
```

## Rollback

```bash
ansible-playbook rollback.yaml \
  -e ansible_become_pass=YOUR_SUDO_PASS \
  -e k8s_rollback_version=1.34.6-1.1
```

## License

MIT — adapt freely for your own cluster.
