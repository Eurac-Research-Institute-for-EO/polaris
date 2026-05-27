# Multinode Installation

This page covers the setup of a multi-node K3s cluster using the `multinode/` directory. The setup provisions a 3-node cluster consisting of one master (control plane) node and two worker nodes, with ArgoCD installed on the master.

This is a **simpler, foundational setup** compared to the full single-node Polaris cluster in `ansible/`. It does not include Sealed Secrets, OIDC/Keycloak integration, CoreDNS overrides, or the app-of-apps ArgoCD pattern — those can be layered on once the cluster is stable.

## Table of Contents
- [Prerequisites](#prerequisites)
  - [Local machine](#local-machine)
  - [Remote nodes](#remote-nodes)
- [Configuration](#configuration)
  - [Inventory](#inventory)
  - [Variables (config.yaml)](#variables-configyaml)
- [Running the playbook](#running-the-playbook)
- [Verifying the cluster](#verifying-the-cluster)
- [Adding workers later](#adding-workers-later)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Local machine

- Ansible installed (`pip install ansible`)
- SSH access to all three nodes
- A user with `sudo` privileges on each node
- The Polaris repository cloned locally

Refer to the [Installation](installation.md) page for detailed steps on setting up SSH keys and Ansible if you haven't done this before.

### Remote nodes

You need **3 nodes** with the following:

| Role | Count | Minimum specs |
|------|-------|---------------|
| Master (control plane) | 1 | 4 CPU, 8 GB RAM, 20 GB disk |
| Worker | 2 | 2 CPU, 4 GB RAM, 20 GB disk |

All nodes must:
- Run a Linux-based OS (Ubuntu recommended)
- Be reachable from your local machine over SSH
- Be able to reach each other over the network (ports `6443` and `10250` must be open between nodes)

**Note:** The master node IP is used by workers to join the cluster, so it must be a stable IP — not a DHCP address that can change between reboots.

---

## Configuration

The `multinode/` directory contains two files you need to edit before running the playbook:

```
multinode/
├── inventory.ini      ← node IPs and roles
├── vars/config.yaml   ← cluster configuration
└── bootstrap.yaml     ← main playbook (no edits needed)
```

### Inventory

Edit `multinode/inventory.ini` to set the IP addresses of your nodes:

```ini
[control]
10.8.244.214

[workers]
10.8.244.215
10.8.244.216

[all_nodes:children]
control
workers

[all_nodes:vars]
ansible_user=ubuntu

[control:vars]
node_role=master

[workers:vars]
node_role=worker
```

- `[control]` — the master node. Only one IP goes here.
- `[workers]` — the worker nodes. Add one IP per line.
- `ansible_user` — the SSH user Ansible will connect as on all nodes. Change this if your user is not `ubuntu`.

If your SSH user differs between the master and workers, you can override it per group:

```ini
[control:vars]
ansible_user=admin

[workers:vars]
ansible_user=ubuntu
```

### Variables (config.yaml)

Edit `multinode/vars/config.yaml` to set cluster-level configuration:

```yaml
cluster:
    domain: "10.8.244.214.nip.io"
    name: "polaris-multinode"
```

| Variable | Description | Example |
|----------|-------------|---------|
| `cluster.domain` | Base domain for cluster services. ArgoCD will be exposed at `argocd.<domain>`. Using [nip.io](https://nip.io) with the master IP is a convenient option for local setups as it resolves to the IP automatically. | `10.8.244.214.nip.io` |
| `cluster.name` | A label for the cluster, used for identification purposes. | `polaris-multinode` |

**Note:** The versions for K3s, ArgoCD, and Helm are set directly in `bootstrap.yaml` under the `control` play's `vars` block. They are pinned to tested versions — only change them if you have a specific reason to.

---

## Running the playbook

From the root of the repository (or any directory — just adjust the paths accordingly):

```bash
ansible-playbook -i multinode/inventory.ini multinode/bootstrap.yaml --ask-become-pass
```

The `--ask-become-pass` flag prompts for the `sudo` password. If your nodes are configured with passwordless sudo, you can omit it.

The playbook runs in three sequential phases:

1. **All nodes** — installs system dependencies (`curl`, `jq`, etc.) on all three nodes in parallel.
2. **Master node** — installs the K3s server, waits for it to be ready, reads the join token, then installs Helm and ArgoCD.
3. **Worker nodes** — installs the K3s agent on each worker using the join token and master IP retrieved from the master play.

You should see the playbook start with:

```
TASK [Print debug message to verify task execution]
ok: [10.8.244.214] => {
    "msg": "Hello Polaris! Playbook version: 0.1"
}
```

And finish with ArgoCD access details printed to the console:

```
TASK [Display ArgoCD access info]
ok: [10.8.244.214] => {
    "msg": [
        "ArgoCD URL: http://argocd.10.8.244.214.nip.io",
        "Username: admin",
        "Password: <generated-password>"
    ]
}
```

**Note:** Save the ArgoCD admin password printed at the end — the initial admin secret is deleted by ArgoCD after first use if you change the password through the UI.

---

## Verifying the cluster

SSH into the master node and verify all nodes have joined:

```bash
sudo kubectl get nodes
```

You should see all three nodes with status `Ready`:

```
NAME        STATUS   ROLES                  AGE   VERSION
master      Ready    control-plane,master   5m    v1.35.0+k3s3
worker-1    Ready    <none>                 3m    v1.35.0+k3s3
worker-2    Ready    <none>                 3m    v1.35.0+k3s3
```

Check that ArgoCD is running:

```bash
sudo kubectl get pods -n argocd
```

All pods should be in `Running` state within a few minutes of the playbook finishing.

To access ArgoCD from your local machine, either:

- Open `http://argocd.<your-domain>` in a browser if your DNS resolves correctly (nip.io handles this automatically).
- Or use port-forwarding as a fallback:

```bash
sudo kubectl port-forward svc/argocd-server -n argocd 8080:80
```

Then open `http://localhost:8080`.

---

## Adding workers later

To add more workers to the cluster after initial setup, add their IPs to `[workers]` in `inventory.ini` and run the playbook with a `--limit` flag to target only the new nodes:

```bash
ansible-playbook -i multinode/inventory.ini multinode/bootstrap.yaml \
  --limit workers \
  --ask-become-pass
```

The worker play will pick up the join token from the master's hostvars automatically, as long as the master is reachable.

---

## Troubleshooting

**Workers fail to join the cluster**

This usually means the master IP is not reachable from the worker nodes on port `6443`. Verify connectivity from a worker:

```bash
curl -k https://<master-ip>:6443
```

You should get an API server response (not a connection refused). If it times out, check firewall rules between the nodes.

**`k3s_node_token` variable undefined during worker play**

This happens if the master play failed or was skipped. The worker play reads the join token from Ansible's hostvars, which is only populated after the master play runs `cat /var/lib/rancher/k3s/server/node-token`. Re-run the full playbook, or run only the master play first:

```bash
ansible-playbook -i multinode/inventory.ini multinode/bootstrap.yaml \
  --limit control \
  --ask-become-pass
```

Then run the full playbook again.

**ArgoCD pod stuck in `Pending`**

K3s ships with a local-path storage provisioner by default, but ArgoCD itself does not require persistent storage in this basic setup. If pods are pending, check node resource availability:

```bash
sudo kubectl describe pod -n argocd <pod-name>
```

Look at the `Events` section for scheduling failures.

**Playbook fails on SSH connection**

Verify that your SSH key is copied to all nodes:

```bash
ssh-copy-id ubuntu@<node-ip>
```

And test the connection manually before running the playbook:

```bash
ssh ubuntu@<node-ip>
```
