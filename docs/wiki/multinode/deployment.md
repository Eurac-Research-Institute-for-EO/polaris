# Deploy & Verify

With [configuration](configuration.md) and [secrets](secrets.md) in place, you're ready to run the main playbook and confirm the cluster came up.

## Running the playbook

First make sure the `polaris` deployment user exists on every node — see [SSH & Deployment User](deployment-user.md). That step runs `playbooks/bootstrap_user.yaml` once and leaves `polaris` with passwordless sudo, which is what `deploy.yaml` connects as.

Then, from the `multinode/` directory:

```bash
ansible-playbook -i inventory.ini playbooks/deploy.yaml
```

No `--ask-become-pass` is needed: the `polaris` deployment user has passwordless sudo. (If you run as a user that needs a sudo password, add `--ask-become-pass`.)

The playbook runs in four sequential phases (each finishes on all its hosts before the next begins, so workers never try to join before the master is up):

1. **All nodes** — installs system dependencies (`curl`, `jq`, etc.) and writes the cluster-wide pod DNS config on all three nodes in parallel.
2. **Master node** — installs the K3s server, waits for it to be ready, reads the join token, then installs Helm, installs ArgoCD, **creates the application secrets** ([Secrets](secrets.md)), and creates the AppProject + app-of-apps.
3. **Worker nodes** — installs the K3s agent on each worker using the join token and master IP retrieved from the master play.
4. **Master node** — waits for every node to report `Ready` and prints `kubectl get nodes`, confirming the workers joined.

ArgoCD access details are printed to the console during phase 2:

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

Confirm the application secrets were created (only the namespaces for the blocks you filled in will exist):

```bash
sudo kubectl get secret grafana-credentials -n monitoring
sudo kubectl get secret -n keycloak
sudo kubectl get secret -n openeo
```

To access ArgoCD from your local machine, either:

- Open `http://argocd.<your-domain>` in a browser if your DNS resolves correctly (nip.io handles this automatically).
- Or use port-forwarding as a fallback:

```bash
sudo kubectl port-forward svc/argocd-server -n argocd 8080:80
```

Then open `http://localhost:8080`.

---

Next: **[Operations](operations.md)**
