# Deployment Guide

This guide provides comprehensive instructions for deploying the Polaris Kubernetes cluster from scratch. It covers all necessary steps including prerequisites, infrastructure setup, application deployment, and verification.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Infrastructure Requirements](#infrastructure-requirements)
3. [Initial Setup](#initial-setup)
4. [Deployment Steps](#deployment-steps)
5. [Post-Deployment Verification](#post-deployment-verification)
6. [Application Management](#application-management)
7. [Troubleshooting](#troubleshooting)

## Prerequisites

Before beginning the deployment, ensure the following requirements are met:

### Software Requirements

- **Ansible**: Version 2.9 or higher installed on your local machine
- **SSH Access**: Passwordless SSH access to the target server
- **Git**: For cloning the repository
- **Ansible Vault Password**: Required for decrypting sensitive configuration

### Access Requirements

- SSH key configured for the target server
- GitHub organization membership in `Eurac-Research-Institute-for-EO`
- GitHub OAuth application credentials (provided via Ansible Vault)

### Network Requirements

- Target server accessible on the local network
- Outbound internet access from the target server for downloading packages
- DNS resolution capability (uses Cloudflare 1.1.1.1 and Google 8.8.8.8)

## Infrastructure Requirements

### Recommended Server Specifications

The base cluster installation requires modest resources. For production use with additional applications, the following specifications are recommended:

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| **CPU** | 2 vCPUs | 4 vCPUs |
| **RAM** | 8 GB | 16 GB |
| **Storage** | 40 GB | 100 GB |
| **Network** | 1 Gbps | 1 Gbps |

**Note**: Actual requirements depend on the operating system, number of applications, and expected workload. The recommended specifications provide headroom for growth and smooth operation.

### Operating System

- **Supported**: Ubuntu 20.04 LTS or later
- **Architecture**: x86_64 (amd64)

## Initial Setup

### 1. Clone the Repository

```bash
git clone https://github.com/Eurac-Research-Institute-for-EO/polaris.git
cd polaris
```

### 2. Configure Inventory

Verify or update the Ansible inventory file at `ansible/inventory.ini`:

```ini
[polaris]
10.8.244.214
```

Replace the IP address with your target server's address if different.

### 3. Verify SSH Access

Test SSH connectivity to the target server:

```bash
ssh <user>@10.8.244.214
```

Or configure your SSH key in `~/.ssh/config`:

```
Host polaris
    HostName 10.8.244.214
    User root
    IdentityFile ~/.ssh/id_rsa
```

## Deployment Steps

### Step 1: Run the Bootstrap Playbook

Check the top `env` block in `ansible/bootstrap.yaml` to ensure all variables are correctly set, especially the GitHub OAuth credentials and RBAC policies. This is the primary configuration for the Ansible playbook and will affect the entire deployment process.

The bootstrap playbook installs and configures all necessary components. Execute the following command from the project root:

```bash
ansible-playbook -i ansible/inventory.ini ansible/bootstrap.yaml --ask-become-pass --ask-vault-pass
```

You will be prompted for the Ansible Vault password and the VM's sudo password. Enter the passwords to decrypt sensitive variables and gain elevated privileges.

### Step 2: Monitor Installation Progress

The bootstrap process includes the following stages:

1. **System Dependencies Installation**
   - Updates package cache
   - Installs curl, apt-transport-https, ca-certificates
   - Configures system repositories

2. **K3s Installation**
   - Downloads and installs K3s v1.35.0+k3s3
   - Initializes single-node Kubernetes cluster
   - Configures kubeconfig at `/etc/rancher/k3s/k3s.yaml`

3. **CoreDNS Configuration**
   - Configures upstream DNS servers (1.1.1.1, 8.8.8.8)
   - Applies custom CoreDNS ConfigMap
   - Restarts CoreDNS pods

4. **Helm Installation**
   - Installs Helm v4.1.1 binary
   - Configures Helm repositories

5. **Sealed Secrets Deployment**
   - Deploys Sealed Secrets controller v0.34.0
   - Configures secret encryption

6. **ArgoCD Installation**
   - Deploys ArgoCD v9.4.0 via Helm
   - Configures Traefik ingress
   - Sets up GitHub OAuth integration
   - Applies RBAC policies

7. **Application Bootstrap**
   - Creates `polaris` AppProject
   - Deploys `polaris-apps` Application (App of Apps pattern)
   - Begins automatic synchronization of applications from Git

The entire process typically takes 5 to 10 minutes depending on network speed and server performance.

### Step 3: Wait for Completion

The playbook will output progress for each task. Wait for the final confirmation message indicating successful completion.

## Post-Deployment Verification

### Verify Cluster Status

SSH into the server and check the cluster status:

```bash
ssh <user>@10.8.244.214
kubectl get nodes
```

Expected output:
```
NAME         STATUS   ROLES                  AGE   VERSION
hostname     Ready    control-plane,master   5m    v1.35.0+k3s3
```

### Verify Core Components

Check that all system components are running:

```bash
kubectl get pods -n kube-system
```

Expected pods:
- coredns
- local-path-provisioner
- metrics-server
- traefik
- sealed-secrets

### Verify ArgoCD Installation

Check ArgoCD components:

```bash
kubectl get pods -n argocd
```

Expected pods:
- argocd-server
- argocd-application-controller
- argocd-repo-server
- argocd-dex-server
- argocd-redis
- argocd-applicationset-controller
- argocd-notifications-controller

All pods should show `Running` status with `1/1` ready state.

### Access ArgoCD UI

Open your browser and navigate to:

```
http://argocd.10.8.244.214.nip.io
```

**Note**: Replace `10.8.244.214` with your server's IP address if different.

### Authenticate with GitHub

1. Click "Log in via GitHub"
2. Authorize the OAuth application
3. You will be redirected back to ArgoCD

Your access level depends on your GitHub account:
- **Admin users** (email-based): Full access to create, modify, and delete applications
- **Organization members**: Read-only access to view applications and their status

The permissions are defined in the RBAC policy applied during installation. They can be modified in the `ansible/bootstrap.yaml` playbook if needed.

### Verify Applications

In the ArgoCD UI, you should see:
- **polaris-apps**: The root App of Apps application
- **hello-nginx**: Example application (if present in the repository)

All applications should show `Healthy` and `Synced` status.

## Application Management

### Understanding the GitOps Workflow

Polaris uses the App of Apps pattern for application management. All applications are defined in the `apps/` directory of the Git repository.

#### Directory Structure

```
apps/
├── kustomization.yaml           # Root kustomization listing all apps
├── hello-nginx-app.yaml         # ArgoCD Application CRD
└── hello-nginx/                 # Application manifests
    ├── kustomization.yaml
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

### Adding a New Application

To deploy a new application to the cluster:

#### 1. Create Application Directory

```bash
mkdir -p apps/myapp
```

#### 2. Define Kubernetes Manifests

Create the necessary Kubernetes resources in the application directory:

```bash
# Example: deployment.yaml, service.yaml, ingress.yaml
```

#### 3. Create Kustomization File

Create `apps/myapp/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp

resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

#### 4. Create ArgoCD Application

Create `apps/myapp-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: polaris
  source:
    repoURL: https://github.com/Eurac-Research-Institute-for-EO/polaris
    targetRevision: main
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

#### 5. Update Root Kustomization

Add the new application to `apps/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - hello-nginx-app.yaml
  - myapp-app.yaml  # Add this line
```

#### 6. Commit and Push

```bash
git add apps/
git commit -m "Add myapp application"
git push origin main
```

#### 7. Automatic Deployment

ArgoCD polls the repository every 3 minutes. Within this time, your application will be automatically deployed. You can also trigger immediate synchronization via the ArgoCD UI or CLI.

### Managing Secrets

Sensitive data must be encrypted using Sealed Secrets before committing to Git.

#### Create a Sealed Secret

```bash
# Create regular secret
kubectl create secret generic mysecret \
  --from-literal=password=mypassword \
  --dry-run=client -o yaml > /tmp/secret.yaml

# Seal the secret
kubeseal -f /tmp/secret.yaml -w apps/myapp/sealed-secret.yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system

# Commit sealed secret to Git
git add apps/myapp/sealed-secret.yaml
git commit -m "Add sealed secret for myapp"
git push
```

The Sealed Secrets controller will automatically decrypt the secret in the cluster.

### Configuring Ingress

To make an application accessible via HTTP:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
    - host: myapp.10.8.244.214.nip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

The application will be accessible at `http://myapp.10.8.244.214.nip.io`.

## Troubleshooting

### ArgoCD Not Accessible

**Symptom**: Unable to access ArgoCD UI at `http://argocd.10.8.244.214.nip.io`

**Solution**:

1. Verify ArgoCD pods are running:
   ```bash
   kubectl get pods -n argocd
   ```

2. Check ingress configuration:
   ```bash
   kubectl get ingress -n argocd
   ```

3. Verify ingress hostname matches your access URL:
   ```bash
   kubectl describe ingress argocd-server -n argocd
   ```

4. Check Traefik logs:
   ```bash
   kubectl logs -n kube-system -l app.kubernetes.io/name=traefik
   ```

### Application Not Syncing

**Symptom**: Application shows `OutOfSync` status in ArgoCD

**Solution**:

1. Check application details in ArgoCD UI for specific errors
2. Verify the Git repository is accessible
3. Check application logs:
   ```bash
   kubectl logs -n argocd deployment/argocd-application-controller
   ```

4. Manually trigger sync via UI or CLI:
   ```bash
   argocd app sync myapp
   ```

### Permission Denied Errors

**Symptom**: Unable to create, modify, or delete applications in ArgoCD

**Solution**:

1. Verify your GitHub email is in the RBAC policy:
   ```bash
   kubectl get configmap argocd-rbac-cm -n argocd -o yaml
   ```

2. Check that `scopes` is set to `[email]`:
   ```yaml
   data:
     scopes: '[email]'
   ```

3. Log out and log back in to ArgoCD to refresh permissions

### DNS Resolution Issues

**Symptom**: Services cannot resolve external domains

**Solution**:

1. Check CoreDNS configuration:
   ```bash
   kubectl get configmap coredns -n kube-system -o yaml
   ```

2. Verify upstream DNS servers are configured:
   ```yaml
   forward . 1.1.1.1 8.8.8.8
   ```

3. Restart CoreDNS:
   ```bash
   kubectl rollout restart deployment/coredns -n kube-system
   ```

### Complete Cluster Reset

If you need to start over completely:

1. Run the cleanup playbook:
   ```bash
   ansible-playbook -i ansible/inventory.ini ansible/cleanup.yaml
   ```

2. Optionally remove K3s entirely:
   ```bash
   ssh root@10.8.244.214
   /usr/local/bin/k3s-uninstall.sh
   ```

3. Re-run the bootstrap playbook:
   ```bash
   ansible-playbook -i ansible/inventory.ini ansible/bootstrap.yaml --ask-vault-pass
   ```

## Additional Resources

- [Architecture Documentation](../architecture.md) - Detailed system architecture diagrams
- [Ansible Setup Guide](./ansible.md) - Detailed Ansible configuration
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/) - Official ArgoCD documentation
- [K3s Documentation](https://docs.k3s.io/) - Official K3s documentation
- [Kustomize Documentation](https://kustomize.io/) - Official Kustomize documentation

## Support

For issues or questions:
- Review the troubleshooting section above
- Check ArgoCD application logs and events
- Consult the GitHub repository issues
- Contact the platform team
