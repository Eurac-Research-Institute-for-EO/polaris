# Troubleshooting

Common failures during a multinode deploy and how to resolve them.

## Workers fail to join the cluster

This usually means the master IP is not reachable from the worker nodes on port `6443`. Verify connectivity from a worker:

```bash
curl -k https://<master-ip>:6443
```

You should get an API server response (not a connection refused). If it times out, check firewall rules between the nodes.

## `k3s_node_token` variable undefined during worker play

This happens if the master play failed or was skipped (e.g. running with `--limit workers`). The worker play reads the join token from Ansible's hostvars, which is only populated after the master play runs `cat /var/lib/rancher/k3s/server/node-token`. Re-run the full playbook, or run only the master play first:

```bash
ansible-playbook -i inventory.ini playbooks/deploy.yaml --limit control
```

Then run the full playbook again.

## ArgoCD pod stuck in `Pending`

K3s ships with a local-path storage provisioner by default, but ArgoCD itself does not require persistent storage in this basic setup. If pods are pending, check node resource availability:

```bash
sudo kubectl describe pod -n argocd <pod-name>
```

Look at the `Events` section for scheduling failures.

## Playbook fails on SSH connection

Verify that your SSH key is copied to all nodes:

```bash
ssh-copy-id ubuntu@<node-ip>
```

And test the connection manually before running the playbook:

```bash
ssh ubuntu@<node-ip>
```

## An app can't find its secret

The application secrets ([Secrets](secrets.md)) are created during the master phase, before the app-of-apps. If an app reports a missing secret, confirm the relevant block was filled in under `secrets:` in `vars/secrets.yaml` and that the secret exists:

```bash
sudo kubectl get secret grafana-credentials -n monitoring -o yaml
```

If the block was added after the first deploy, re-run the deploy (or just the secrets task) — `kubectl apply` is idempotent and will create what's missing:

```bash
ansible-playbook -i inventory.ini playbooks/deploy.yaml --limit control
```

## Pods resolve external names to `0.0.0.0` (e.g. ArgoCD can't reach the Keycloak issuer)

The corporate DNS domains (`eurac.edu`, `unibz.it`, `scientificnet.org`) answer *unknown* names with a `0.0.0.0` wildcard instead of `NXDOMAIN`. Pods default to `ndots:5` and inherit those domains as search suffixes, so an external name like `edp-portal.eurac.edu` gets tried as `edp-portal.eurac.edu.unibz.it` → `0.0.0.0` *before* the real lookup. Two task files address this:

- `tasks/common/dns.yaml` points kubelet at a managed `/etc/rancher/k3s/resolv.conf` (via k3s `--resolv-conf`) with **no corporate search domains** (`cluster.dns_search` is empty), so every pod resolves full names correctly through CoreDNS. This is the cluster-wide fix.
- `tasks/master/coredns.yaml` forwards CoreDNS to `cluster.dns_servers` (`10.8.28.2` first, the server that answers from the pod network).

Verify resolution from a throwaway pod (should return a real IP, never `0.0.0.0`):

```bash
sudo kubectl run dnscheck --image=debian:stable-slim --rm -i --restart=Never \
  --command -- getent hosts edp-portal.eurac.edu
```

If a pod still has corporate search domains in `/etc/resolv.conf`, it predates the fix — restart that workload so it picks up the new node config.
