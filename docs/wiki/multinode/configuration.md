# Configuration

You edit three files before deploying: `inventory.ini` (which nodes), `vars/config.yaml` (non-secret settings), and `vars/secrets.yaml` (covered separately in [Secrets](secrets.md)). The playbooks themselves need no changes.

## Inventory

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

## Variables (config.yaml)

`multinode/vars/config.yaml` is the **single source of inputs** — every playbook and task file reads its configurables from here. Edit it to match your environment:

```yaml
cluster:
    domain: "10.8.244.214.nip.io"
    name: "polaris-multinode"

initial_user: jzvolensky
deploy_user: polaris
deploy_user_pubkey: "~/.ssh/id_ed_polaris.pub"

versions:
    k3s: "v1.35.0+k3s3"
    helm: "v4.1.1"
    argocd: "9.4.0"

kubeconfig_path: /etc/rancher/k3s/k3s.yaml
```

| Variable | Description | Example |
|----------|-------------|---------|
| `cluster.domain` | Base domain for cluster services. ArgoCD is exposed at `argocd.<domain>`. Using [nip.io](https://nip.io) with the master IP resolves to the IP automatically for local setups. | `10.8.244.214.nip.io` |
| `cluster.name` | A label for the cluster, used for identification purposes. | `polaris-multinode` |
| `initial_user` | Initial VM login user, used only by `bootstrap_user.yaml` to create the deployment user. See [SSH & Deployment User](deployment-user.md). | `jzvolensky` |
| `deploy_user` | Dedicated deployment user created on every node; the steady-state `ansible_user`. | `polaris` |
| `deploy_user_pubkey` | Local path to the public key installed for the deployment user (`~` is expanded). | `~/.ssh/id_ed_polaris.pub` |
| `versions.k3s` / `versions.helm` / `versions.argocd` | Pinned, tested component versions. Change only with a specific reason. | `v1.35.0+k3s3` |
| `kubeconfig_path` | Path to the K3s-generated kubeconfig on the master. | `/etc/rancher/k3s/k3s.yaml` |

The `config.yaml` also has an `argocd:` block reused from the single-node setup:

- `argocd.oidc` — Keycloak SSO: `issuer`, `client_id` (e.g. `polaris_argocd`), `name`, and `requested_scopes`. Set `enabled: false` to skip SSO and use local admin only. The **client secret is not here** — see [Secrets](secrets.md).
- `argocd.rbac` — `admin_group` (Keycloak group granted `role:admin`, e.g. `polaris_admins`) and `default_role` for other authenticated users.
- `argocd.project` — the `polaris` AppProject: `source_repos` (allowed repos), `destinations`, and `cluster_resource_whitelist`.
- `argocd.app_of_apps` — the app-of-apps Application: `repo_url`, `target_revision`, `path`, and `sync_policy`.

---

Next: **[Secrets](secrets.md)**
