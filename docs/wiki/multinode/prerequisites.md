# Prerequisites

Before you start, make sure your local machine and the target nodes meet the requirements below.

## Local machine

- Ansible installed (`pip install ansible`)
- SSH access to all three nodes
- A user with `sudo` privileges on each node
- The Polaris repository cloned locally

Refer to the [Installation](../installation.md) page for detailed steps on setting up SSH keys and Ansible if you haven't done this before.

## Remote nodes

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

Next: **[SSH & Deployment User](deployment-user.md)**
