# ArgoCD Image Updater

[ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/) watches the
container registries behind your images and, when a newer image appears, updates
the tag an ArgoCD `Application` deploys — so a new build rolls out without anyone
editing manifests by hand.

In this cluster it is installed on the master **alongside ArgoCD** (it is ArgoCD
platform infra, not an app workload) and is wired to update the **openeo dev API
image**, committing each bump back to the `polaris-apps` repo so the change is
visible in git and revertable.

!!! note "It does nothing until you opt an app in"
    Image Updater only touches `Application`s that carry
    `argocd-image-updater.argoproj.io/*` annotations. Installing it is inert by
    default; only the openeo **dev** overlay is annotated.

## How it is deployed

`tasks/master/argocd/image-updater.yaml` installs it via Helm right after ArgoCD,
pinned to a chart version in `vars/config.yaml`:

```yaml
# vars/config.yaml
versions:
    argocd_image_updater: "1.2.4"   # argo-helm CHART version (ships appVersion v1.2.2)

argocd:
    image_updater:
        enabled: true               # set false to skip the install entirely
        log_level: "info"
```

It runs in the `argocd` namespace in **`kubernetes` applications-api mode** — it
patches the `Application` resources directly via the Kubernetes API, so it needs
no ArgoCD API token. Set `enabled: false` to leave it out.

## Write-back: how updates reach git

When a tracked image changes, Image Updater **commits the new tag** to
`polaris-apps` (the `dev` branch) rather than only patching the live cluster
object. This matters here because the cluster runs the **app-of-apps** pattern
with `selfHeal: true`: if Image Updater patched the live `Application`, the parent
app would revert it as drift. Writing to git makes git the source of truth, so
there is no fight — and you get history and easy rollback.

The openeo Application is **multi-source** (Helm chart from the `charts` repo,
values from `polaris-apps` via a `$values` ref) specifically so the image tag can
live in `polaris-apps` where Image Updater can write it:

```
apps/openeo-argoworkflows/values/image-dev.yaml   ← Image Updater writes image.tag here
```

A bump produces a commit authored by `argocd-image-updater`. To **revert**,
`git revert` that commit (or edit the tag back and push); ArgoCD re-syncs the
previous image.

## GitHub setup (one-time, reproducible)

Git write-back pushes to `polaris-apps` over SSH, so it needs a **deploy key with
write access**. Do this once.

### 1. Generate a dedicated keypair

On the master node (or wherever you keep `secrets.yaml`):

```bash
# No passphrase (-N "") — Image Updater cannot unlock a protected key.
ssh-keygen -t ed25519 -C "polaris-image-updater" \
  -f ~/.ssh/polaris_image_updater -N ""
```

### 2. Register the public key as a deploy key

Add the **public** key to the `polaris-apps` repo with write access:

=== "GitHub UI"

    Repo → **Settings → Deploy keys → Add deploy key**. Paste
    `~/.ssh/polaris_image_updater.pub`, tick **Allow write access**, save.

=== "gh CLI"

    ```bash
    gh repo deploy-key add ~/.ssh/polaris_image_updater.pub \
      -R Eurac-Research-Institute-for-EO/polaris-apps -w -t image-updater
    ```

!!! warning "Write access is required"
    A read-only deploy key lets ArgoCD *read* the repo but Image Updater cannot
    push its commits. Make sure **Allow write access** is enabled.

### 3. Put the private key in `secrets.yaml`

Paste the **private** key into `vars/secrets.yaml` as a block scalar (mind the
indentation — every key line is indented under the `|`):

```yaml
# vars/secrets.yaml
image_updater_git_ssh_key: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXktdjEAAAAA...
  ...
  -----END OPENSSH PRIVATE KEY-----
```

`tasks/master/secrets.yaml` reads this and creates the Secret
`argocd/polaris-apps-image-updater` (key `sshPrivateKey`). Leave the value empty
to skip it — git write-back stays disabled until it is filled in.

!!! note "github.com host keys are already handled"
    The `argo-cd` chart ships `argocd-ssh-known-hosts-cm` with github.com's host
    keys, and the Image Updater chart mounts it automatically — no `known_hosts`
    setup needed.

### 4. (Optional) clean up

Once the private key is in `secrets.yaml`, the key files are no longer needed on
the node:

```bash
rm ~/.ssh/polaris_image_updater ~/.ssh/polaris_image_updater.pub
```

The only copies that matter are the gitignored `secrets.yaml` and the in-cluster
Secret. Re-run `playbooks/deploy.yaml` (or just the master play) to apply.

## How tracking works (the workflow)

There is **no central list** of tracked apps. Image Updater discovers work by
reading the annotations on every ArgoCD `Application`, so tracking an app is
entirely about adding annotations to *that app*:

```
new image pushed → controller (every ~2 min) reads the app's annotations →
finds a newer tag per the strategy → writes the new tag to a file in polaris-apps
(git commit on the dev branch) → ArgoCD sees the commit and re-syncs the app
```

The only global setup — the install, the SSH write-back key, the commit author —
is already done. Everything below is per-application.

## Add tracking to an application

### 1. Find the image and how its manifest sets it

Two questions decide which annotations you need:

- **Helm or Kustomize app?**
- If Helm: is the image a **split** `repository` + `tag` (two values), or a
  **single combined** `repo:tag` string?

### 2. Make the tag git-writable

Image Updater commits the new tag to git, so the tag must live somewhere ArgoCD
reads **from `polaris-apps`** (the repo the SSH deploy key can push to):

- **App sourced directly from `polaris-apps`** (plain manifests / kustomize, e.g.
  `hello-nginx`): already writable — write-back commits straight into the app's
  files.
- **App whose chart lives in another repo** (e.g. openeo's chart is in `charts`):
  use the **multi-source `$values`** pattern so the image value lives in a
  `polaris-apps` values file. openeo does this — see its
  `base/helm-openeo-argoworkflows.yaml` and `values/image-dev.yaml`.

### 3. Add the annotations

Put them on the `Application`'s `metadata.annotations`. In this repo apps are
defined in `polaris-apps`, so add them via the overlay's kustomize patch (as the
openeo dev overlay does) or directly on the Application manifest.

The building blocks (replace `<alias>` with a short name you pick per image):

| Annotation (prefix `argocd-image-updater.argoproj.io/`) | Purpose |
|---|---|
| `image-list` | `<alias>=<registry>/<image>[:<tag>]`, comma-separated for multiple images |
| `<alias>.update-strategy` | `digest` / `semver` / `alphabetical` / `newest-build` |
| `<alias>.allow-tags` | optional regex filter, e.g. `regexp:^v[0-9]+\.[0-9]+\.[0-9]+$` |
| `<alias>.helm.image-name` / `.helm.image-tag` | Helm: the **split** value keys to write |
| `<alias>.helm.image-spec` | Helm: the **single combined** value key (see caveat) |
| `<alias>.kustomize.image-name` | Kustomize: the image name to override |
| `write-back-method` | `git:secret:argocd/polaris-apps-image-updater` (use the deploy key) |
| `git-repository` / `git-branch` | SSH URL of `polaris-apps` and the branch (`dev`) |
| `write-back-target` | `helmvalues:<path>` or `kustomization[:<path>]` — the file to edit |

Pick the image mapping by app shape:

=== "Helm, split repository + tag (recommended)"

    Works perfectly with git write-back. This is the openeo API image:

    ```yaml
    argocd-image-updater.argoproj.io/image-list: "api=ghcr.io/ORG/myapp:dev"
    argocd-image-updater.argoproj.io/api.update-strategy: "digest"
    argocd-image-updater.argoproj.io/api.helm.image-name: "image.repository"
    argocd-image-updater.argoproj.io/api.helm.image-tag: "image.tag"
    argocd-image-updater.argoproj.io/write-back-method: "git:secret:argocd/polaris-apps-image-updater"
    argocd-image-updater.argoproj.io/git-repository: "git@github.com:Eurac-Research-Institute-for-EO/polaris-apps.git"
    argocd-image-updater.argoproj.io/git-branch: "dev"
    argocd-image-updater.argoproj.io/write-back-target: "helmvalues:/apps/myapp/values/image-dev.yaml"
    ```

=== "Kustomize app"

    For an app whose source is `polaris-apps` directly:

    ```yaml
    argocd-image-updater.argoproj.io/image-list: "web=ghcr.io/ORG/web:dev"
    argocd-image-updater.argoproj.io/web.update-strategy: "digest"
    argocd-image-updater.argoproj.io/web.kustomize.image-name: "ghcr.io/ORG/web"
    argocd-image-updater.argoproj.io/write-back-method: "git:secret:argocd/polaris-apps-image-updater"
    argocd-image-updater.argoproj.io/git-branch: "dev"
    argocd-image-updater.argoproj.io/write-back-target: "kustomization"
    ```

=== "Helm, combined image string ⚠️"

    `<alias>.helm.image-spec` points at one value holding the full `repo:tag`. With
    **git** write-back in our `valueFiles`-based apps this drops the tag (writes the
    repo only) — a broken reference. Avoid it; either get the chart to expose a
    split `repository`/`tag`, or keep that image pinned manually. This is exactly
    why the openeo **executor** image is not auto-tracked.

### 4. Commit, push, watch

Commit the annotation change to `polaris-apps` `dev` and push. Within ~2 min the
controller logs should mention the app; on a new image it commits the tag and
ArgoCD re-syncs. See [Verifying it works](#verifying-it-works) below.

## Update strategies

The openeo dev app uses **`digest`**, which tracks a single **mutable** tag
(`:dev`) and fires whenever a new image is pushed to it — no version scheme
required. It writes `image.tag: dev@sha256:<digest>`, a valid digest-pinned
reference.

Switch the strategy to match how your CI tags images:

| Your tags | `update-strategy` | Notes |
|-----------|-------------------|-------|
| mutable `:dev` / `:latest` | `digest` | current default; redeploys on any new push to that tag |
| semver `v1.4.2` | `semver` | add `<alias>.allow-tags: "regexp:^v?[0-9]+\.[0-9]+\.[0-9]+$"` |
| calver `2026-06-29` | `alphabetical` | highest lexical tag wins |
| commit-SHA | `newest-build` | newest by image build time; add an `allow-tags` regex |

## Reverting a bad update

Because every bump is a git commit, rollback is a git operation:

```bash
cd polaris-apps
git log --oneline -- apps/<app>/values/image-dev.yaml   # find the argocd-image-updater commit
git revert <commit>
git push origin dev
```

ArgoCD re-syncs the previous tag. (You can also just edit the tag back and push.)

## Verifying it works

```bash
# Controller is running (the chart names the Deployment ...-controller):
kubectl get deploy -n argocd argocd-image-updater-controller

# What it is doing (it logs each app it considers and any update):
kubectl logs -n argocd deploy/argocd-image-updater-controller -f

# The write-back credential exists:
kubectl get secret -n argocd polaris-apps-image-updater
```

After a new `:dev` image is pushed, within the poll interval (~2 min) you should
see a commit from `argocd-image-updater` on the `polaris-apps` `dev` branch
touching `values/image-dev.yaml`, followed by an ArgoCD re-sync of the dev app.

If nothing happens, check the controller logs first — the most common causes are a
missing/empty `image_updater_git_ssh_key`, a deploy key without **write** access,
or an `update-strategy` that does not match how your images are tagged.

---

Next: **[Infrastructure Manager (IM)](im.md)**
