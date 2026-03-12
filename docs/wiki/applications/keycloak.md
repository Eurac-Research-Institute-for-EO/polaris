# Keycloak Deployment Guide

This guide provides step-by-step instructions for deploying Keycloak as an identity and access management solution in the Polaris cluster.

**NOTE**: *This is a guide covering the provided template for Keycloak. However, at Eurac it is recommended to use the existing internal Keycloak instance. If in any case you want to deploy a new Keycloak instance, please follow the instructions below.*

## Overview

This deployment includes:
- PostgreSQL 16 database (1 replica, 10GB storage)
- Keycloak 23.0 identity provider (2 replicas)
- Traefik ingress for external access
- Sealed Secrets for credential management

## Prerequisites

Before starting, ensure:
- Polaris cluster is running with K3s
- Sealed Secrets controller is installed and operational
- At least 3-4GB free RAM and 1-2 CPU cores available
- 10GB storage available for PostgreSQL

Verify Sealed Secrets is running:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

## Deployment Steps

### Step 1: Generate Secure Passwords

Generate strong random passwords for the database and admin account:

```bash
# Generate database password
DB_PASSWORD=$(openssl rand -base64 32)

# Generate Keycloak admin password
ADMIN_PASSWORD=$(openssl rand -base64 32)

# Display passwords (save these securely!)
echo "Database Password: $DB_PASSWORD"
echo "Keycloak Admin Password: $ADMIN_PASSWORD"

# Optionally, save to a secure file
cat > /tmp/keycloak-passwords.txt <<EOF
Database Password: $DB_PASSWORD
Keycloak Admin Password: $ADMIN_PASSWORD
Generated: $(date)
EOF
chmod 600 /tmp/keycloak-passwords.txt
```

**IMPORTANT**: Save these passwords securely. You will need the admin password to access the Keycloak console.

### Step 2: Create PostgreSQL Sealed Secret

Create the sealed secret for PostgreSQL credentials:

```bash
kubectl create secret generic postgresql-credentials \
  --namespace=keycloak \
  --from-literal=POSTGRES_USER=keycloak \
  --from-literal=POSTGRES_PASSWORD="$DB_PASSWORD" \
  --from-literal=POSTGRES_DB=keycloak \
  --dry-run=client -o yaml | \
kubeseal --format=yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  > postgresql-sealed-secret.yaml
```

Verify the sealed secret was created:

```bash
cat postgresql-sealed-secret.yaml
```

### Step 3: Create Keycloak Sealed Secret

Create the sealed secret for Keycloak credentials (using the same database password):

```bash
kubectl create secret generic keycloak-credentials \
  --namespace=keycloak \
  --from-literal=KEYCLOAK_ADMIN=admin \
  --from-literal=KEYCLOAK_ADMIN_PASSWORD="$ADMIN_PASSWORD" \
  --from-literal=KC_DB_URL=jdbc:postgresql://postgresql:5432/keycloak \
  --from-literal=KC_DB_USERNAME=keycloak \
  --from-literal=KC_DB_PASSWORD="$DB_PASSWORD" \
  --dry-run=client -o yaml | \
kubeseal --format=yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  > keycloak-sealed-secret.yaml
```

Verify the sealed secret was created:

```bash
cat keycloak-sealed-secret.yaml
```

### Step 4: Update Kustomization

Update the kustomization file to use sealed secrets instead of placeholder secrets:

```bash
# Edit apps/keycloak/kustomization.yaml
# Replace these lines:
#   - postgresql-secret.yaml
#   - keycloak-secret.yaml
# With:
#   - postgresql-sealed-secret.yaml
#   - keycloak-sealed-secret.yaml
```

Or use sed to automate it:

```bash
sed -i 's/postgresql-secret.yaml/postgresql-sealed-secret.yaml/' apps/keycloak/kustomization.yaml
sed -i 's/keycloak-secret.yaml/keycloak-sealed-secret.yaml/' apps/keycloak/kustomization.yaml
```

Verify the changes:

```bash
grep sealed apps/keycloak/kustomization.yaml
```

### Step 5: Monitor Deployment

ArgoCD will automatically detect the changes and begin deployment. Monitor the progress:

```bash
# Watch ArgoCD application status
watch kubectl get application keycloak -n argocd

# Watch pod creation
watch kubectl get pods -n keycloak

# View detailed events
kubectl get events -n keycloak --sort-by='.lastTimestamp'
```

Expected timeline:
- 0-30s: Namespace and PVC created
- 30s-2m: PostgreSQL StatefulSet starting
- 2m-3m: PostgreSQL ready, Keycloak Deployment starting
- 3m-5m: Keycloak pods ready and healthy

### Step 6: Verify Deployment

Check all components are running:

```bash
# Check all pods are ready
kubectl get pods -n keycloak

# Expected output:
# NAME                        READY   STATUS    RESTARTS   AGE
# postgresql-0                1/1     Running   0          3m
# keycloak-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
# keycloak-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

kubectl get svc -n keycloak

kubectl get ingress -n keycloak
```

All pods should show `1/1` Ready and `Running` status.

### Step 7: Access Keycloak

Open your browser and navigate to:

```
http://keycloak.10.8.244.214.nip.io
```

**Note**: Replace `10.8.244.214` with your cluster's IP address if different.

You should see the Keycloak welcome page.

## Initial Configuration

### Access Admin Console

1. Click "Administration Console" on the welcome page
2. Login with:
   - **Username**: `admin`
   - **Password**: (use the `$ADMIN_PASSWORD` you generated in Step 1)

If you saved passwords to a file:
```bash
cat /tmp/keycloak-passwords.txt
```

### Create Application Realm

The master realm is for Keycloak administration only. Create a separate realm for your applications:

1. Hover over "Master" in the top-left corner
2. Click "Create Realm"
3. Enter realm name: `polaris`
4. Click "Create"

### Configure GitHub Identity Provider (Optional)

To allow users to login with GitHub:

#### 1. Create GitHub OAuth Application

1. Go to: https://github.com/organizations/Eurac-Research-Institute-for-EO/settings/applications
2. Click "New OAuth App"
3. Fill in:
   - **Application name**: `Keycloak - Polaris`
   - **Homepage URL**: `http://keycloak.10.8.244.214.nip.io/realms/polaris`
   - **Authorization callback URL**: `http://keycloak.10.8.244.214.nip.io/realms/polaris/broker/github/endpoint`
4. Click "Register application"
5. Copy the **Client ID**
6. Click "Generate a new client secret"
7. Copy the **Client Secret**

#### 2. Configure in Keycloak

1. In Keycloak admin console, select the `polaris` realm
2. Go to "Identity providers" in the left menu
3. Click "Add provider" and select "GitHub"
4. Fill in:
   - **Client ID**: (paste from GitHub)
   - **Client Secret**: (paste from GitHub)
5. Click "Add"

Now users can login to Keycloak using their GitHub accounts.

### Create Groups

Create groups for role-based access control:

1. In the `polaris` realm, go to "Groups"
2. Click "Create group"
3. Create the following groups:
   - `admins` (full access)
   - `developers` (read-write access)
   - `viewers` (read-only access)

### Assign Users to Groups

After users login via GitHub:

1. Go to "Users" in the left menu
2. Click on a user
3. Go to "Groups" tab
4. Click "Join Group"
5. Select appropriate group (e.g., `admins`)
6. Click "Join"

## Integrating with ArgoCD

To use Keycloak for ArgoCD authentication:

### 1. Create ArgoCD Client in Keycloak

1. In the `polaris` realm, go to "Clients"
2. Click "Create client"
3. Fill in:
   - **Client ID**: `argocd`
   - **Client type**: `OpenID Connect`
4. Click "Next"
5. Enable "Client authentication"
6. Set **Valid redirect URIs**: `http://argocd.10.8.244.214.nip.io/auth/callback`
7. Click "Save"
8. Go to "Credentials" tab
9. Copy the **Client Secret** (save this securely)

### 2. Create Group Mapper

1. In the ArgoCD client, go to "Client scopes"
2. Click on `argocd-dedicated`
3. Click "Add mapper" → "By configuration"
4. Select "Group Membership"
5. Configure:
   - **Name**: `groups`
   - **Token Claim Name**: `groups`
   - **Full group path**: OFF
6. Click "Save"

### 3. Update ArgoCD Configuration

Update ArgoCD to use Keycloak OIDC:

```bash
kubectl edit configmap argocd-cm -n argocd
```

Replace the `dex.config` section with:

```yaml
data:
  url: "http://argocd.10.8.244.214.nip.io"
  dex.config: |
    connectors:
      - type: oidc
        id: keycloak
        name: Keycloak
        config:
          issuer: http://keycloak.10.8.244.214.nip.io/realms/polaris
          clientID: argocd
          clientSecret: <paste-client-secret-here>
          redirectURI: http://argocd.10.8.244.214.nip.io/api/dex/callback
          scopes:
            - openid
            - profile
            - email
            - groups
```

### 4. Update ArgoCD RBAC

Update RBAC to use Keycloak groups:

```bash
kubectl edit configmap argocd-rbac-cm -n argocd
```

Update the policy:

```yaml
data:
  policy.csv: |
    g, /admins, role:admin
    g, /developers, role:developer
    g, /viewers, role:readonly
  policy.default: role:readonly
  scopes: '[groups, email]'
```

### 5. Restart ArgoCD

Restart ArgoCD to apply changes:

```bash
kubectl rollout restart deployment/argocd-server -n argocd
kubectl rollout status deployment/argocd-server -n argocd
```

### 6. Test Authentication

1. Logout of ArgoCD
2. Navigate to: `http://argocd.10.8.244.214.nip.io`
3. Click "Log in via Keycloak"
4. You should be redirected to Keycloak
5. Login with GitHub (or local user)
6. You should be redirected back to ArgoCD with appropriate permissions

## Backup and Recovery

### Backup PostgreSQL Database

Create a backup of the Keycloak database:

```bash
mkdir -p ~/keycloak-backups

kubectl exec -n keycloak postgresql-0 -- \
  pg_dump -U keycloak keycloak \
  > ~/keycloak-backups/keycloak-backup-$(date +%Y%m%d-%H%M%S).sql

gzip ~/keycloak-backups/keycloak-backup-*.sql
```

### Restore from Backup

To restore a backup:

```bash
gunzip ~/keycloak-backups/keycloak-backup-20260304-120000.sql.gz

kubectl exec -i -n keycloak postgresql-0 -- \
  psql -U keycloak keycloak < ~/keycloak-backups/keycloak-backup-20260304-120000.sql
```

### Automated Backup Script

Create a cron job for daily backups:

```bash
cat > ~/backup-keycloak.sh <<'EOF'
#!/bin/bash
BACKUP_DIR="$HOME/keycloak-backups"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/keycloak-backup-$DATE.sql"

mkdir -p "$BACKUP_DIR"

kubectl exec -n keycloak postgresql-0 -- \
  pg_dump -U keycloak keycloak > "$BACKUP_FILE"

gzip "$BACKUP_FILE"

# Keep only last 7 days of backups
find "$BACKUP_DIR" -name "keycloak-backup-*.sql.gz" -mtime +7 -delete

echo "Backup completed: $BACKUP_FILE.gz"
EOF

chmod +x ~/backup-keycloak.sh

(crontab -l 2>/dev/null; echo "0 2 * * * $HOME/backup-keycloak.sh") | crontab -
```

## Troubleshooting

### PostgreSQL Not Starting

Check PostgreSQL logs:

```bash
kubectl logs -n keycloak postgresql-0
```

Common issues:
- PVC not bound: Check storage availability
- Permission errors: Check PVC permissions
- Previous data corruption: Delete PVC and redeploy

### Keycloak Pods CrashLooping

Check Keycloak logs:

```bash
kubectl logs -n keycloak deployment/keycloak
```

Common issues:
- Database not ready: Wait for PostgreSQL to be healthy
- Wrong database credentials: Verify sealed secrets match
- Memory limits too low: Increase resource limits

### Cannot Access Keycloak UI

Check ingress configuration:

```bash
kubectl get ingress -n keycloak
kubectl describe ingress keycloak -n keycloak
```

Verify Traefik is routing correctly:

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik
```

Test internal connectivity:

```bash
kubectl run curl-test --image=curlimages/curl:latest --rm -i --restart=Never -- \
  curl -I http://keycloak.keycloak.svc.cluster.local:8080
```

### Forgot Admin Password

Reset the admin password:

```bash
# Generate new password
NEW_ADMIN_PASSWORD=$(openssl rand -base64 32)
echo "New Admin Password: $NEW_ADMIN_PASSWORD"

# Get current database password
DB_PASSWORD=$(kubectl get secret postgresql-credentials -n keycloak -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d)

# Create new sealed secret
kubectl create secret generic keycloak-credentials \
  --namespace=keycloak \
  --from-literal=KEYCLOAK_ADMIN=admin \
  --from-literal=KEYCLOAK_ADMIN_PASSWORD="$NEW_ADMIN_PASSWORD" \
  --from-literal=KC_DB_URL=jdbc:postgresql://postgresql:5432/keycloak \
  --from-literal=KC_DB_USERNAME=keycloak \
  --from-literal=KC_DB_PASSWORD="$DB_PASSWORD" \
  --dry-run=client -o yaml | \
kubeseal --format=yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system | \
kubectl apply -f -

# Restart Keycloak
kubectl rollout restart deployment/keycloak -n keycloak
kubectl rollout status deployment/keycloak -n keycloak
```

Wait 30 seconds for pods to restart, then try the new password.

## Monitoring

### Health Checks

Check Keycloak health endpoints:

```bash
# Health check
curl http://keycloak.10.8.244.214.nip.io/health

# Readiness
curl http://keycloak.10.8.244.214.nip.io/health/ready

# Liveness
curl http://keycloak.10.8.244.214.nip.io/health/live
```

### View Logs

```bash
# Keycloak logs
kubectl logs -n keycloak deployment/keycloak -f

# PostgreSQL logs
kubectl logs -n keycloak postgresql-0 -f

# All logs in namespace
kubectl logs -n keycloak --all-containers=true -f
```

### Resource Usage

```bash
# View resource consumption
kubectl top pods -n keycloak

# Detailed pod information
kubectl describe pod -n keycloak -l app=keycloak
```

## Maintenance

### Upgrade Keycloak

To upgrade to a new version:

1. Check release notes: https://www.keycloak.org/docs/latest/release_notes/
2. Update image version in `apps/keycloak/keycloak-deployment.yaml`:
   ```yaml
   image: quay.io/keycloak/keycloak:24.0  # Update version
   ```
3. Commit and push changes
4. ArgoCD will perform rolling update
5. Monitor the upgrade:
   ```bash
   kubectl rollout status deployment/keycloak -n keycloak
   ```

### Scale Keycloak

To increase number of replicas:

1. Edit `apps/keycloak/kustomization.yaml`:
   ```yaml
   replicas:
     - name: keycloak
       count: 3  # Increase from 2
   ```
2. Commit and push
3. ArgoCD will scale automatically

## Security Best Practices

1. **Use strong passwords**: Always generate random passwords (minimum 32 characters)
2. **Never commit secrets**: Only commit sealed secrets to Git
3. **Regular backups**: Automate daily PostgreSQL backups
4. **Enable HTTPS**: Add TLS certificates for production use
5. **Update regularly**: Keep Keycloak and PostgreSQL updated
6. **Monitor logs**: Review Keycloak audit logs regularly
7. **Limit access**: Use network policies to restrict PostgreSQL access
8. **MFA**: Enable multi-factor authentication in Keycloak for admin accounts

## Resource Requirements

Expected resource consumption:

| Component | CPU | Memory | Storage |
|-----------|-----|--------|---------|
| Keycloak (per pod) | 0.5-1 core | 1-2 GB | - |
| PostgreSQL | 0.25-0.5 core | 0.5-1 GB | 10 GB |
| **Total (2 Keycloak replicas)** | **1.25-2.5 cores** | **3-5 GB** | **10 GB** |

## Additional Resources

- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [Keycloak Admin Guide](https://www.keycloak.org/docs/latest/server_admin/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [OpenID Connect Protocol](https://openid.net/connect/)
- [Sealed Secrets Documentation](https://github.com/bitnami-labs/sealed-secrets)

## Support

For issues or questions:
1. Check the troubleshooting section above
2. Review Keycloak logs: `kubectl logs -n keycloak deployment/keycloak`
3. Check PostgreSQL logs: `kubectl logs -n keycloak postgresql-0`
4. Consult official Keycloak documentation
5. Open an issue in the GitHub repository
