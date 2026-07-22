# Infrastructure Manager (IM)

[Infrastructure Manager](https://imdocs.readthedocs.io/) (IM, from GRyCAP) deploys
container and VM applications — described as TOSCA templates — onto this cluster
and onto cloud providers, with a web **dashboard** to drive and manage them.

It is installed on the master via **Helm by ansible** (`tasks/master/im.yaml`),
the same way as ArgoCD and Image Updater — *not* through ArgoCD. The reason is
secret hygiene: the IM chart renders the dashboard's OIDC client secret and its
`credentials_key` straight from helm values into a ConfigMap/Secret (it has no
`existingSecret` option), so an ArgoCD deploy would commit those to git. Driving
it from ansible lets both come from the gitignored `vars/secrets.yaml`.

!!! note "IM's deployments are outside ArgoCD"
    Applications that IM itself deploys are plain Kubernetes resources that no
    ArgoCD Application tracks, so there is no prune/self-heal clash with the
    GitOps apps. You create and manage them from the IM dashboard.

## How it is deployed

Non-secret config lives in `vars/config.yaml`, pinned to a chart version:

```yaml
# vars/config.yaml
versions:
    im: "1.8.0"                 # grycap/IM chart version

im:
    enabled: true              # set false to skip the install
    ingress_host: "im.10.8.244.253.nip.io"
    oidc:
        name: "Keycloak"
        base_url: "https://edp-portal.eurac.edu/auth/realms/edp"
        client_id: "polaris_im"
        scopes: "openid email profile"
        group_membership: "[]"  # JSON array of allowed Keycloak groups; [] = any user
    support_email: "..."
```

The task adds the `grycap` Helm repo, renders a values file (traefik Ingress, the
Gateway API `httproute` disabled, dashboard enabled, MySQL persistent), runs
`helm upgrade --install im grycap/IM`, and waits on the `im-backend` Deployment.
Everything lands in the `im` namespace.

## Setup (one-time)

### 1. Create the Keycloak client

In the `edp` realm, create a **confidential** client:

- **Client ID:** `polaris_im`
- **Client authentication:** ON
- **Valid redirect URIs:** `http://im.10.8.244.253.nip.io/*`

!!! warning "Use a wildcard redirect URI to start"
    The dashboard's exact OIDC callback path isn't documented, so register the
    wildcard above first. If login fails, `kubectl logs -n im deploy/im-dashboard`
    shows the `redirect_uri` it actually sent — then you can tighten it.

### 2. Fill in the secrets

In `vars/secrets.yaml` (gitignored), under the top-level `im_secrets` key:

```yaml
im_secrets:
    oidc_client_secret: "<the polaris_im client secret>"
    credentials_key: "<openssl rand -base64 32>"
```

`credentials_key` is the Fernet key IM uses to encrypt stored cloud credentials in
its database — generate a fresh one, do **not** reuse the chart default:

```bash
openssl rand -base64 32
```

### 3. Deploy

```bash
cd multinode
ansible-playbook -i inventory.ini playbooks/deploy.yaml
```

(`im_secrets` is a separate top-level key from the config `im:` block on purpose —
both files are loaded together, and a shared key name would collide.)

## Access

- **API:** `http://im.10.8.244.253.nip.io/im`
- **Dashboard:** `http://im.10.8.244.253.nip.io/im-dashboard` (log in via Keycloak)

## Deploying to this cluster (the Kubernetes connector)

IM targets an existing cluster with a credential you add **in the dashboard** —
no chart change needed:

```
host = https://10.8.244.253:6443
token = <ServiceAccount token>
```

plus an optional default `namespace`. IM then creates the app's resources
(Deployments, Services, Ingress, PVCs) from a TOSCA template.

!!! note "ServiceAccount + RBAC + token — TODO"
    The token above comes from a dedicated ServiceAccount with RBAC to create the
    resources you want IM to manage (scoped as tightly as you like). That piece is
    not set up yet — it is a planned follow-up.

## Verifying

```bash
kubectl get pods -n im
kubectl rollout status -n im deploy/im-backend
kubectl rollout status -n im deploy/im-dashboard
kubectl logs -n im deploy/im-dashboard --tail=50      # OIDC / redirect_uri issues show here
```

## Caveats

- **No TLS** — served as plain HTTP over nip.io, like the other dev endpoints. Add
  TLS before any real use (OIDC over HTTP is fine for testing only).
- **MySQL** is left at the chart's internal default (never exposed) to avoid
  mis-wiring its Bitnami subchart — a hardening follow-up.
- **OIDC redirect path** — see the wildcard-redirect warning above.

---

Next: **[Troubleshooting](troubleshooting.md)**
