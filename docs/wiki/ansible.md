# Ansible Setup

Infrastructure setup for polaris is done using Ansible. The Ansible playbooks are located in the `ansible` directory of the project.

## Prerequisites

- Ansible installed on your local machine.
- SSH access to the target servers where you want to deploy Polaris.
- Inventory file configured with the target hosts.
- Connecting from a machine that has access to the target servers (VPN or on-premises network).

## Inventory File

The `inventory.ini` file should contain the details of the target hosts. For example:

```ini
[polaris]
<ip_address_of_target_server>
```

## Playbook Configuration

The `bootstrap.yaml` file contains the main playbook to install and configure the necessary components for Polaris. You can customize this playbook to fit your specific requirements, such as adding additional tasks or modifying existing ones.

The `tasks` directory contains individual task files that are included in the main playbook. These tasks handle specific aspects of the setup, such as installing Argo CD, configuring secrets, and setting up applications.

| Task file | Description |
|------|-------------|
| `bootstrap.yaml` | Main playbook to set up base Polaris infrastructure. |
| `tasks/system_deps.yaml` | Task to install system dependencies on the target server. |
| `tasks/k3s.yaml` | Task to install k3s on the target server. |
| `tasks/helm.yaml` | Task to install Helm on the target server. |
| `tasks/coredns.yaml` | Task to configure CoreDNS on the target server. |
| `tasks/sealed_secrets.yaml` | Task to install Sealed Secrets on the target server. |
| `tasks/argocd.yaml` | Task to install Argo CD on the target server. |
| `tasks/argo_github_auth.yaml` | Task to configure Argo CD with GitHub OAuth. |

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

## Running the Playbook

To run the Ansible playbook, navigate to the `ansible` directory and execute the following command:

```bash
ansible-playbook -i inventory.ini bootstrap.yaml --ask-become-pass --ask-vault-pass
```
