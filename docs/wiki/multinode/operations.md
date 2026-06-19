# Operations

Day-two tasks: scaling the cluster out, or wiping it to start fresh.

## Adding workers later

To add more workers after initial setup, add their IPs to `[workers]` in `inventory.ini`, then run the playbook scoped to the master **and** the new workers:

```bash
ansible-playbook -i inventory.ini playbooks/deploy.yaml \
  --limit control,<new-worker-ip>
```

The `control` node must be in scope because the worker play reads the K3s join token from the master's hostvars, which is only populated when the master play runs. The master's tasks are idempotent (K3s is already installed, so they're skipped) — they just re-expose the token for the new workers to join.

## Tearing down / starting over

To wipe the cluster from all nodes and start fresh, run `uninstall.yaml`:

```bash
ansible-playbook -i inventory.ini playbooks/uninstall.yaml
```

It uninstalls the K3s agents on the workers and the K3s server on the master. Because ArgoCD and every workload run *inside* K3s, uninstalling the server removes them all in one step — no separate ArgoCD cleanup needed. It also removes the Helm binary and leftover `/tmp` manifests.

It **keeps** the system libraries, the `polaris` deployment user, and SSH access — so you can immediately rebuild with `playbooks/deploy.yaml` without redoing [user setup](deployment-user.md).

The playbook prompts for confirmation (it's destructive and irreversible). To skip the prompt in scripts:

```bash
ansible-playbook -i inventory.ini playbooks/uninstall.yaml -e teardown_confirmed=yes
```

It's also safe to re-run on already-clean nodes — each step is skipped if the relevant uninstall script isn't present.

---

Next: **[Troubleshooting](troubleshooting.md)**
