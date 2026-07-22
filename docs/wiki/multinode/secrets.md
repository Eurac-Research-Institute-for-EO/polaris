# Secrets

All secrets live in a single gitignored file, `multinode/vars/secrets.yaml`. It holds two things:

1. The **ArgoCD OIDC client secret**, used when `argocd.oidc.enabled` is true.
2. An optional **`secrets:` block** of application secrets that `tasks/master/secrets.yaml` applies to the cluster before ArgoCD first syncs.

Create it from the committed template and fill in the values:

```bash
cd multinode
cp vars/secrets.example.yaml vars/secrets.yaml
$EDITOR vars/secrets.yaml
```

`secrets.yaml` is in `.gitignore` — never commit it. If it's missing when OIDC is enabled, the playbook stops early with a message telling you to create it.

```yaml
# vars/secrets.yaml

# ArgoCD Keycloak OIDC client secret (config.yaml: argocd.oidc.client_id).
oidc_client_secret: "<the polaris_argocd client secret>"

# Application secrets — see the section below.
secrets:
    grafana:
        admin_user: "admin"
        admin_password: "CHANGEME"
        oidc_client_secret: "CHANGEME"
    keycloak:
        admin_user: "admin"
        admin_password: "CHANGEME"
        db_name: "keycloak"
        db_username: "keycloak"
        db_password: "CHANGEME"
        db_host: "postgresql.keycloak.svc.cluster.local"
        db_port: 5432
    openeo:
        db_name: "postgres"
        db_username: "postgres"
        db_password: "CHANGEME"
        redis_password: "CHANGEME"
```

## How application secrets work

`tasks/master/secrets.yaml` runs on the master during the deploy, after ArgoCD is installed but **before** the app-of-apps is created. For each block present under `secrets:`, it creates the target namespace and applies a plain Kubernetes `Secret` (idempotently, via `kubectl apply`). Because the secrets exist before ArgoCD syncs the apps repo, each application finds the secret it references by name on its first sync — no manual step, no ordering race.

Every block is **optional**. Drop a section to skip creating its secrets — e.g. keep only `grafana` to deploy monitoring alone, and add `keycloak` / `openeo` later.

| Block | Namespace | Secret(s) created | Keys | Consumed by |
|-------|-----------|-------------------|------|-------------|
| `grafana` | `monitoring` | `grafana-credentials` | `admin-user`, `admin-password`, `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET` | kube-prometheus-stack Grafana (`existingSecret: grafana-credentials`) |
| `keycloak` | `keycloak` | `keycloak-credentials` | `KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD`, `KC_DB_URL`, `KC_DB_USERNAME`, `KC_DB_PASSWORD` | Keycloak deployment |
| `keycloak` | `keycloak` | `postgresql-credentials` | `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | Keycloak's PostgreSQL |
| `openeo` | `openeo-dev` / `openeo` | `<release>-postgresql` (`openeo-dev-postgresql` / `openeo-postgresql`) | `postgres-password` | OpenEO Postgres (Bitnami subchart + API/cronjobs, via `postgresql.auth.existingSecret`) |
| `openeo` | `openeo-dev` / `openeo` | `openeo-redis` | `REDIS_PASSWORD` | **No-op** — the chart runs Redis without auth and never reads this; skipped while `redis_password` is empty |

`KC_DB_URL` is assembled from the keycloak block as `jdbc:postgresql://<db_host>:<db_port>/<db_name>`.

## ArgoCD Image Updater SSH key (optional)

`secrets.yaml` also holds an optional top-level `image_updater_git_ssh_key` — the SSH deploy key ArgoCD Image Updater uses to push image-tag bumps back to the `polaris-apps` repo. It is created as the Secret `argocd/polaris-apps-image-updater` and is skipped while empty. Generating the key and registering the GitHub deploy key is covered in **[ArgoCD Image Updater](image-updater.md)**.

## Why plain Secrets, not Sealed Secrets?

This setup deliberately uses **plain** Kubernetes Secrets applied by Ansible, rather than the Sealed Secrets controller. The reasoning:

- **In-cluster, the two are identical.** A Sealed Secret is encrypted only at rest in git; the controller decrypts it into an ordinary `Secret` stored base64-encoded in etcd — exactly what a plain `kubectl apply` produces. Sealing buys nothing for in-cluster secrecy.
- **Nothing is committed to git here.** The plaintext already lives in `vars/secrets.yaml` on the operator's machine, and Ansible applies it directly. Sealing's only real benefit — safe-to-commit ciphertext — does not apply, so it would add a controller and a `kubeseal` step for no gain.
- **You re-supply config on every deploy anyway.** Rebuilds read the same `secrets.yaml`, so there's no value in persisting an encrypted copy in the repo.

If you ever need encryption **at rest in etcd** (a different concern from sealing), enable K3s secrets encryption (`--secrets-encryption`); it protects every Secret regardless of how it was created. And if the apps later move to a pure-GitOps model where encrypted secrets are committed to the apps repo, that's when Sealed Secrets earns its place — at which point the controller would be added back.

---

Next: **[Deploy & Verify](deployment.md)**
