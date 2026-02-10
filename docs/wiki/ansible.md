# Ansible Setup

Infrastructure setup for polaris is done using Ansible. The Ansible playbooks are located in the `ansible` directory of the project.

## Prerequisites

- Ansible installed on your local machine.
- SSH access to the target servers where you want to deploy Polaris.
- Inventory file configured with the target hosts.
- Connecting from a machine that has access to the target servers (VPN or on-premises network).

## Running the Playbook

To run the Ansible playbook, navigate to the `ansible` directory and execute the following command:

```bash
ansible-playbook -i inventory.ini bootstrap.yaml
```

## Inventory File

The `inventory.ini` file should contain the details of the target hosts. For example:

```ini
[polaris]
<ip_address_of_target_server>
```

## Playbook Configuration

The `bootstrap.yaml` file contains the tasks to set up Polaris on the target servers. You can customize the playbook according to your requirements.

Currently, the playbook includes tasks for installing necessary dependencies, configuring the environment, and deploying Polaris base cluster with Argo CD.

Installed components include:

- Helm
- K3s
- Sealed Secrets
- ArgoCD

## Configure Argo CD repository

By default we point the Argo CD application to the `main` branch of the Polaris repository. If you want to use a different branch, or add more repositories, you can modify the Argo CD application manifest located in the `argocd` directory.

Here is a sample configuration for the Argo CD application:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: polaris
  namespace: argocd
spec:
  description: Polaris Work Cluster Applications
  sourceRepos:
    - 'https://github.com/Eurac-Research-Institute-for-EO/polaris'
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
---
```

## Secrets Management

Sensitive information such as GitHub OAuth client ID and secret are stored using Ansible Vault. Make sure to create and edit the vault file to include these secrets before running the playbook.

To use GitHub authentication we create our organization application and generate a client ID and secret. These values are then encrypted using Ansible Vault and stored in the `bootstrap.yaml` file as variables.

In GitHub settings make sure to set the callback URL to `https://<ARGOCD_SERVER_URL>/api/dex/callback` where `<ARGOCD_SERVER_URL>` is the URL of your ArgoCD server.

For example:

- Application name: Polaris ArgoCD (or any name you prefer)
- Homepage URL: `http://argocd.10.8.244.214.nip.io`
- Authorization callback URL: `http://argocd.10.8.244.214.nip.io/api/dex/callback`

```sh
cd ansible
ansible-vault encrypt_string 'YOUR_CLIENT_ID' --name 'github_oauth_client_id'
ansible-vault encrypt_string 'YOUR_CLIENT_SECRET' --name 'github_oauth_client_secret'
```

Replace the encrypted values in the `bootstrap.yaml` file with the output from the above commands. These are then used in the playbook to create sealed secrets for ArgoCD to enable GitHub authentication.

**Note**: You will be prompted to enter a vault password, to be able to retrieve the values later. If you forget it, just recreate the vault with a new password and update the encrypted values in the `bootstrap.yaml` file.

These encrypted values are safe to commit.
