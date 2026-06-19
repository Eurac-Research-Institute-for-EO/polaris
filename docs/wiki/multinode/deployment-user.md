# SSH & Deployment User Setup

Before Ansible can configure the cluster, two things must be true:

1. Your local machine can SSH into all three VMs **without a password** (key-based auth).
2. Every node has a **dedicated `polaris` user** that all deployment playbooks run as.

This page covers the whole flow end to end. The good news: once your SSH key is on the nodes, creating and verifying the `polaris` user is a **single `ansible-playbook` command** — no editing the inventory between steps, no re-running anything by hand.

## The two-user model

| | Initial VM user | Deployment user |
|---|---|---|
| Username | `<your_eurac_user>` | `polaris` |
| Purpose | First login, bootstrap only | All Ansible playbook runs |
| Exists from | The VM image / provider | Created by `bootstrap_user.yaml` |
| Sudo | Needs sudo (to create `polaris`) | Passwordless sudo |

We don't run deployments as a personal login account. A dedicated `polaris` user keeps deployment activity separate from interactive admin, gives clear ownership of what Ansible created, and ends up identical on every node regardless of how the base image was provisioned.

`bootstrap_user.yaml` connects as `<your_eurac_user>` **only for its first play** (to create `polaris`); every other playbook — including the verification play it chains to — connects as `polaris`. That's why the inventory can permanently set `ansible_user=polaris`.

> **Note:** The IPs (`10.8.244.253`, `10.8.244.185`, `10.8.244.73`) come from `multinode/inventory.ini`. The two usernames come from `multinode/vars/config.yaml` (`initial_user` and `deploy_user`). Use your own values where they differ — `<your_eurac_user>` is whatever your personal login on the VMs is.

## Table of Contents
- [The flow at a glance](#the-flow-at-a-glance)
- [Step 1 — Generate an SSH key](#step-1-generate-an-ssh-key)
- [Step 2 — Copy your key to the initial user on all nodes](#step-2-copy-your-key-to-the-initial-user-on-all-nodes)
- [Step 3 — Configure `config.yaml` and the inventory](#step-3-configure-configyaml-and-the-inventory)
- [Step 4 — Run the bootstrap-user playbook (creates + verifies)](#step-4-run-the-bootstrap-user-playbook-creates-verifies)
- [What the playbook does](#what-the-playbook-does)
- [Re-checking access later](#re-checking-access-later)
- [Troubleshooting](#troubleshooting)

---

## The flow at a glance

1. **Generate** an SSH key pair locally (if you don't have one).
2. **`ssh-copy-id`** your public key to the **initial user** on all nodes — the only step that uses a password, and it's one-time.
3. **Configure** `config.yaml` (the two usernames) and `inventory.ini` (key path).
4. **Run `bootstrap_user.yaml` once** — it creates `polaris` on every node (as the initial user) and then verifies access as `polaris`, in a single command.

That's it. From then on, every playbook connects as `polaris` with passwordless sudo.

---

## Step 1 — Generate an SSH key

Check whether you already have a key:

```bash
ls ~/.ssh/*.pub 2>/dev/null
```

If you need one, generate an Ed25519 key with a dedicated name so it's clearly tied to this project:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed_polaris -C "polaris-multinode"
```

This creates:

- `~/.ssh/id_ed_polaris` — your **private** key. Never copy this to the nodes.
- `~/.ssh/id_ed_polaris.pub` — your **public** key. This is what gets installed on the nodes.

> **Important — non-default key names:** The SSH client only auto-offers the standard names (`id_rsa`, `id_ed25519`, …). A custom name like `id_ed_polaris` will **not** be presented automatically, so you must point at it explicitly: with `-i` for `ssh-copy-id`/`ssh`, and with `ansible_ssh_private_key_file` for Ansible (both covered below). If you used the default `~/.ssh/id_ed25519`, you can drop the `-i` flags and the key-file line.

---

## Step 2 — Copy your key to the initial user on all nodes

`ssh-copy-id` appends your public key to `~/.ssh/authorized_keys` on the remote node. It prompts for the node's password **once** per node — after that, key-based auth takes over. Replace `<your_eurac_user>` with your login.

**All three in a loop:**

```bash
for ip in 10.8.244.253 10.8.244.185 10.8.244.73; do
  ssh-copy-id -i ~/.ssh/id_ed_polaris.pub <your_eurac_user>@$ip
done
```

**Or one node at a time** (useful if one fails and you want to retry just that one):

```bash
ssh-copy-id -i ~/.ssh/id_ed_polaris.pub <your_eurac_user>@10.8.244.253
ssh-copy-id -i ~/.ssh/id_ed_polaris.pub <your_eurac_user>@10.8.244.185
ssh-copy-id -i ~/.ssh/id_ed_polaris.pub <your_eurac_user>@10.8.244.73
```

> **Host-key prompts:** On first contact SSH asks you to confirm each node's fingerprint. In a `for` loop this prompt can fail (`ssh-copy-id` opens two connections per host), so if a node errors with `Host key verification failed`, pre-seed it and retry that node individually:
> ```bash
> ssh-keyscan -H 10.8.244.185 >> ~/.ssh/known_hosts
> ssh-copy-id -i ~/.ssh/id_ed_polaris.pub <your_eurac_user>@10.8.244.185
> ```

Verify key-based login works on all three (`-i` selects the custom key, `BatchMode=yes` disables password fallback so you get a clean pass/fail):

```bash
for ip in 10.8.244.253 10.8.244.185 10.8.244.73; do
  echo "== $ip =="; ssh -i ~/.ssh/id_ed_polaris -o BatchMode=yes <your_eurac_user>@$ip hostname
done
```

Each node should print its hostname. If any node prompts for a password or errors, see [Troubleshooting](#troubleshooting) — the initial user must have a home directory **and** sudo on every node before continuing.

---

## Step 3 — Configure `config.yaml` and the inventory

**`multinode/vars/config.yaml`** — set the usernames and the key to install:

```yaml
# Used ONLY to bootstrap the deployment user. Must already have your SSH key
# and sudo on every node.
initial_user: <your_eurac_user>

# The deployment user, created on every node. All playbooks run as this.
deploy_user: polaris

# Local path to the public key installed for the deployment user (~ expanded).
deploy_user_pubkey: "~/.ssh/id_ed_polaris.pub"
```

**`multinode/inventory.ini`** — `ansible_user` is already the **steady-state** deployment user (`polaris`); you don't change it before/after bootstrap. For a non-default key name, point Ansible at your private key:

```ini
[all_nodes:vars]
ansible_user=polaris
ansible_ssh_private_key_file=~/.ssh/id_ed_polaris
```

> **Why `ansible_ssh_private_key_file`?** Same reason as the `-i` flags above — Ansible won't offer a non-default key name on its own. Without this line you'll hit `Permission denied (publickey)` even though the key is installed. (The same `id_ed_polaris` key is used for both users, since `bootstrap_user.yaml` installs it for `polaris` too.)
>
> **Gotcha — inventory `ansible_user` beats both `-u` and `remote_user`:** `ansible_user` set in the inventory **overrides** the `-u` command-line flag *and* a play's `remote_user:` keyword. That's why `bootstrap_user.yaml` switches to the initial user with a play **variable** (`ansible_user: "{{ initial_user }}"`, which sits higher in precedence) rather than the `remote_user` keyword. Set users via the inventory and `config.yaml`, not `-u`.

The playbook installs the key at `deploy_user_pubkey` (above) into the new `polaris` user. To use a different key for one run without editing `config.yaml`, override it: `-e deploy_user_pubkey=/path/to/key.pub`.

---

## Step 4 — Run the bootstrap-user playbook (creates + verifies)

A single command does everything — creates `polaris` on all nodes and then verifies it works:

```bash
ansible-playbook -i multinode/inventory.ini multinode/playbooks/bootstrap_user.yaml --ask-become-pass
```

- No `-u`, no inventory edits — Play 1 connects as `initial_user` from `config.yaml`, the rest as `polaris`.
- `--ask-become-pass` — prompts once for the **initial user's** sudo password (needed to create `polaris`). `polaris` itself gets passwordless sudo, so later playbooks won't need this flag.

The run finishes with the verification play (the same checks as `ping.yaml`) reporting `OK` per node:

```
TASK [Report access status]
ok: [10.8.244.253] => {
    "changed": false,
    "msg": "OK — connected as 'polaris', sudo resolves to 'root'."
}
```

When all three report `OK`, the deployment user is ready and you can continue with [Configuration](configuration.md), then [Deploy & Verify](deployment.md). The playbook is **idempotent** — safe to re-run if anything was interrupted.

---

## What the playbook does

`multinode/playbooks/bootstrap_user.yaml` is two plays:

**Play 1 — create the user** (connects as `initial_user`, `become: true`):

1. **Creates the user** (`ansible.builtin.user`) — `polaris`, with a home directory, a bash shell, and membership in the `sudo` group.
2. **Installs the SSH key** (`ansible.posix.authorized_key`) — adds your public key to `polaris`'s `~/.ssh/authorized_keys`.
3. **Grants passwordless sudo** (`ansible.builtin.copy`) — writes `/etc/sudoers.d/polaris` with `polaris ALL=(ALL) NOPASSWD:ALL`, validated by `visudo -cf` before being applied.
4. **Allows the user through `sshd`** (`ansible.builtin.lineinfile`) — these VMs restrict logins with an `AllowUsers` line in `/etc/ssh/sshd_config`. If that line exists and doesn't already list `polaris`, the task appends it (validated with `sshd -t`) and a handler reloads `sshd`. It's a no-op if there's no `AllowUsers` line or `polaris` is already there.

**Play 2 — verify** (`import_playbook: ping.yaml`, connects as `polaris`): pings each node, confirms `whoami` is `polaris`, and asserts passwordless sudo resolves to `root`.

Passwordless sudo is needed because Ansible runs unattended; without it every play would require `--ask-become-pass`. The `visudo` validation guards against writing a broken sudoers file.

---

## Re-checking access later

To re-verify the deployment user at any time without re-creating it, run the verification play on its own:

```bash
ansible-playbook -i multinode/inventory.ini multinode/playbooks/ping.yaml
```

It connects as `polaris` (from the inventory) and runs the same ping + sudo assertion per node.

---

## Troubleshooting

**`Missing sudo password` during the verify play**

`polaris` exists but its passwordless-sudo drop-in (`/etc/sudoers.d/polaris`) isn't in effect — usually Play 1 didn't complete. Just re-run the whole playbook (it's idempotent); the `Grant passwordless sudo` task will report `changed` once it lands. To inspect the live state on a node:
```bash
ssh -i ~/.ssh/id_ed_polaris <your_eurac_user>@10.8.244.253 \
  'sudo cat /etc/sudoers.d/polaris; echo ---; sudo -l -U polaris'
```

**`Permission denied (publickey)` connecting as `polaris`, but the key is installed correctly**

The node likely restricts logins with an `AllowUsers` (or `AllowGroups`) line in `sshd_config` that doesn't list `polaris` — `sshd` then rejects the user before the key is even checked. Play 1 handles this automatically, but to inspect or fix by hand:
```bash
ssh -t -i ~/.ssh/id_ed_polaris <your_eurac_user>@<node-ip> \
  'sudo grep -RiE "^(AllowUsers|AllowGroups)" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/'
```
If `polaris` is missing from an `AllowUsers` line, add it and reload: `sudo systemctl reload ssh`.

**`Permission denied (publickey)` for any user, key freshly installed**

If it's not an `AllowUsers` issue, check the non-default key name is being offered: `ansible_ssh_private_key_file=~/.ssh/id_ed_polaris` in the inventory, `ssh -i ~/.ssh/id_ed_polaris …` for manual tests, or `ssh-add ~/.ssh/id_ed_polaris`. Also verify home/`.ssh`/`authorized_keys` ownership and permissions (`700`/`600`, owned by the user).

**Ansible connects as the wrong user**

`ansible_user` in the inventory overrides `-u`. Fix the values in `inventory.ini` / `config.yaml` rather than passing `-u`.

**`Host key verification failed` during `ssh-copy-id`**

Pre-seed the node's host key and retry just that node:
```bash
ssh-keyscan -H <node-ip> >> ~/.ssh/known_hosts
```

**`Could not chdir to home directory … No such file or directory` / `mkdir: .ssh: Permission denied`**

The initial user exists on that node but has **no home directory** (inconsistent provisioning). Create it from a privileged account:
```bash
ssh -t <admin-user>@<node-ip> 'sudo mkhomedir_helper <your_eurac_user>'
```

**`<your_eurac_user> is not in the sudoers file`**

The initial user lacks sudo on that node, so it can't create `polaris` there. You need a privileged account on that node — try `ubuntu` or `root`, or use your VM provider's console to add the user to the `sudo` group:
```bash
ssh ubuntu@<node-ip> 'id; sudo -n -l 2>&1 | head -2'
ssh root@<node-ip> 'echo root-login-works'
```
If a different account (e.g. `ubuntu`) is the consistent admin across nodes, set `initial_user` to it in `config.yaml` for the bootstrap step.

---

Next: **[Configuration](configuration.md)**
