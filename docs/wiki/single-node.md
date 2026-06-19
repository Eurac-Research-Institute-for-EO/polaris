# Single Node Overview

This section covers the full Polaris cluster on a **single remote server**, deployed from the `ansible/` directory. One K3s node runs both the control plane and the workloads, with ArgoCD managing the application set.

This is the original Polaris setup. If you instead want a 3-node cluster (one master + two workers), see the [Multinode](multinode/multinode.md) section, which is a simpler, foundational variant.

## What you'll do

1. **[Installation](installation.md)** — prerequisites, configure `inventory.ini` and `vars/`, and run the `bootstrap.yaml` playbook end to end.
2. **[Secrets](secrets.md)** — manage application secrets via `cluster_secrets.yaml`, and update them on a running cluster.
3. **[Uninstall](uninstall.md)** — tear everything down with `cleanup.yaml` to start from scratch.

## How it works

The whole installation is driven by Ansible from your local machine over SSH:

- `inventory.ini` — the remote server(s) and the SSH user Ansible connects as.
- `bootstrap.yaml` — the main playbook: installs K3s, Helm, and ArgoCD, applies the application secrets, and syncs the app-of-apps.
- `secrets.yaml` — updates secrets on an already-installed cluster without a full re-run.
- `cleanup.yaml` — removes ArgoCD and K3s to reset the node.

You only edit `inventory.ini` and the files under `vars/`; the playbooks themselves need no changes.

---

Next: **[Installation](installation.md)**
