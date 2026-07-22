# Multinode Overview

This section covers setting up a multi-node K3s cluster from the `multinode/` directory. The setup provisions a 3-node cluster — one master (control plane) and two workers — with ArgoCD installed on the master and the application secrets it needs created up front.

It is a **simpler, foundational setup** than the full single-node Polaris cluster in `ansible/`. It reuses the same ArgoCD setup — Keycloak OIDC single sign-on, the `polaris` AppProject, and the app-of-apps Application — and configures DNS so pods resolve corporate names correctly (CoreDNS upstream resolvers + a cluster-wide kubelet resolv.conf — see [Troubleshooting](troubleshooting.md)).

Secrets are kept simple: instead of running the Sealed Secrets controller, the playbook applies plain Kubernetes Secrets imperatively from a gitignored `vars/secrets.yaml` before ArgoCD first syncs, so each app finds the secret it references. See [Secrets](secrets.md) for the reasoning.

## Deploy, start to finish

Work through the chapters in order — each ends with a pointer to the next:

1. **[Prerequisites](prerequisites.md)** — what you need on your machine and the nodes.
2. **[SSH & Deployment User](deployment-user.md)** — create the `polaris` deployment user on every node (run once).
3. **[Configuration](configuration.md)** — edit `inventory.ini` and `vars/config.yaml`.
4. **[Secrets](secrets.md)** — create `vars/secrets.yaml` (OIDC + application secrets).
5. **[Deploy & Verify](deployment.md)** — run `playbooks/deploy.yaml` and confirm the cluster.
6. **[Operations](operations.md)** — add workers later, or tear down and start over.
7. **[ArgoCD Image Updater](image-updater.md)** — auto-update image tags with git write-back (optional).
8. **[Infrastructure Manager (IM)](im.md)** — TOSCA-based app deployment with a dashboard (optional).
9. **[Troubleshooting](troubleshooting.md)** — DNS, joins, and other common issues.

## Directory layout

The `multinode/` directory is organized into playbooks, reusable task files, and config:

```
multinode/
├── inventory.ini              ← node IPs, roles, SSH user (edit)
├── vars/
│   ├── config.yaml            ← all non-secret configuration (edit)
│   ├── secrets.example.yaml   ← template for secrets (committed)
│   └── secrets.yaml           ← OIDC + application secrets (gitignored; you create it)
├── playbooks/
│   ├── bootstrap_user.yaml    ← creates the polaris deployment user (run first)
│   ├── ping.yaml              ← verifies deployment-user access
│   ├── deploy.yaml            ← main playbook: system deps, DNS, K3s, ArgoCD, secrets
│   └── uninstall.yaml         ← removes K3s + ArgoCD from all nodes (start over)
└── tasks/
    ├── common/                ← runs on all nodes
    │   ├── packages.yaml      ← apt dependencies
    │   └── dns.yaml           ← cluster-wide pod DNS (kubelet resolv.conf)
    ├── master/
    │   ├── k3s-server.yaml    ← K3s server + join token
    │   ├── coredns.yaml       ← CoreDNS upstream resolvers
    │   ├── helm.yaml          ← Helm binary
    │   ├── secrets.yaml       ← application Secrets (grafana, keycloak, openeo, image-updater key)
    │   ├── im.yaml            ← Infrastructure Manager (IM) via Helm
    │   └── argocd/
    │       ├── install.yaml       ← ArgoCD via Helm (ingress + OIDC config)
    │       ├── oidc.yaml          ← patch Keycloak client secret, restart
    │       ├── image-updater.yaml ← ArgoCD Image Updater via Helm
    │       └── project.yaml       ← AppProject + app-of-apps
    └── worker/
        └── k3s-agent.yaml     ← K3s agent join
```

The task files are small and single-purpose; `playbooks/deploy.yaml` is the orchestrator that includes them in the right order across the master/worker phases. You only edit `inventory.ini`, `vars/config.yaml`, and `vars/secrets.yaml`; the playbooks need no changes.

---

Next: **[Prerequisites](prerequisites.md)**
