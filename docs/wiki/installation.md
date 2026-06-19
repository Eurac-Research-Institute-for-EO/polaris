# Installation

This page provides a step by step overview of how to install Polaris

Most of the setup is completely automated, but it requires a few manual steps and careful configuration.

## Table of Contents
- [Prerequisites](#prerequisites)
   - [Local machine](#local-machine)
        - [Ansible installation](#ansible-installation)
        - [SSH access](#ssh-access)
   - [Remote server](#remote-server)
- [Configuration of Ansible](#configuration-of-ansible)

## Prerequisites

There are two sets of prerequisites as we can use Ansible from our local machine to install Polaris on a remote server.

### Local machine

- Ansible
- SSH access to the remote server
- A user with sudo privileges on the remote server
- Clone the Polaris repository to your local machine

#### Ansible installation

To install Ansible on your local machine, you can use pip:

```bash
pip install ansible
```

Have a look at the [Ansible Documentation](https://docs.ansible.com/#get_started) for more details.

#### SSH access

Ansible requires SSH access to the remote server. You can set up SSH keys for passwordless authentication, which is recommended for Ansible. You can generate an SSH key pair using the following command:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Then copy the public key to the remote server using:
```bash
ssh-copy-id user@remote_server_ip
```

Test the SSH connection to ensure it works without prompting for a password:
```bash
ssh user@remote_server_ip
``` 

**Note: Replace `user` and `remote_server_ip` with your actual username and the IP address of your remote server.**

### Remote server

- A Linux-based operating system (e.g., Ubuntu)
- Sufficient resources (CPU, RAM, disk space) to run Polaris

As a best practice it is recommended to create a dedicated user for Polaris installation and management. This user should have sudo privileges to allow Ansible to perform necessary tasks during the installation process.

Please ask the VM administrator to set this up for you if you do not have the necessary permissions to create users on the remote server.

The specific resource requirements will depend on the expected workload and usage of Polaris. It is recommended to have at least 4 CPU cores, 8 GB of RAM, and 20 GB of disk space for a basic installation. If you ask the VM administrator nicely, they might be able to provide you with a server that meets these requirements.

## Configuration of Ansible

Now is a good time to clone the Polaris repository to your local machine if you haven't already:

```bash
git clone https://github.com/Eurac-Research-Institute-for-EO/polaris.git
```

In there, you will see a bunch of things, but the most important one for now is the `ansible` directory. This is where all the Ansible playbooks and configuration files are located.

Inside `/ansible` we have the:

- `inventory.ini` file: This is where you will define your inventory of hosts (remote servers) that Ansible will manage.
- `bootstrap.yaml` file: This is the main playbook that will be executed to install Polaris on the remote server.
- `cleanup.yaml` file: This playbook can be used to clean up any resources created during the installation process if needed.
- `tasks/` directory: This directory contains individual task files that are included in the main playbooks.
- `vars/` directory: This directory contains variable files that can be used to customize the installation process.

### Inventory configuration

The `inventory.ini` file is where you will define the remote server(s) that Ansible will manage. You can specify the IP address or hostname of the remote server, along with the user that Ansible should use to connect.

Here is an example of how to configure the `inventory.ini` file:

```ini
[polaris]
10.8.244.214 ansible_connection=ssh ansible_user=polaris_user
```

Where:
- `<ip_address>` is the IP address of your remote server. This can also be multiple servers if you want to manage more than one.
- `ansible_connection=ssh` tells Ansible to use SSH to connect to the remote server.
- `ansible_user=polaris_user` specifies the user that Ansible should use to connect to the remote server. This should be the user with sudo privileges that you set up earlier.

### Variable configuration

The `vars/` directory contains variable file that can be used to customize the installation process. You can create a new variable file or edit the existing ones to set specific values for your installation.

In theory these variables can be anything, in our case it is mostly related to the secrets of the different apps and Argo CD itself. These secrets are used to configure the applications that will be installed as part of Polaris.

We provide an example `cluster_secrets.example.yaml` that shows all of the currently available configurations.

**Note: The playbook `boostrap.yaml` itself contains some set variables such as versions for Argo CD, K3s, Helm etc. It is only recommended to touch these if you know what you are doing. The versions provided are ones that we have tested and changing them may cause issues in the automatic deployment.**

### Running Ansible playbook

Now that we have configured our inventory and variables, we can run the Ansible playbook to install Polaris on the remote server.

To run the playbook, use the following command from the `ansible` directory (or wherever just make sure to provide the correct path to the playbook and inventory file):

```bash
ansible-playbook -i inventory.ini bootstrap.yaml --ask-become-pass
```

This command will execute the `bootstrap.yaml` playbook using the inventory defined in `inventory.ini`. The `--ask-become-pass` flag will prompt you for the sudo password of the user specified in the inventory file, which is necessary for Ansible to perform tasks that require elevated privileges on the remote server. In case the user you are using for Ansible has passwordless sudo privileges, you can omit the `--ask-become-pass` flag.

You will now see Ansible executing the tasks defined in the playbook, and if everything is configured correctly, Polaris should be installed on the remote server by the end of the playbook execution.

We have added certain debug messages to make it clear that the process has started and finished

Assuming you started the playbook correctly you will see:

```bash
"Hello Polaris! Playbook version: <version_number>"
```

At the end you will see something like:

```bash
ArgoCD Applications Deployment Status
Total Applications: <number_of_applications>
Applications Status:
- <app_name_1>: <app_status_1>
- <app_name_2>: <app_status_2>
- <app_name_3>: <app_status_3>
- ...
Polaris-Apps Sync Status: <sync_status>
Sync Conditions: <sync_conditions>
```

## Troubleshooting

In case you encounter some issues during the installation process, here are a few tips to help you troubleshoot:

- Check the Ansible output for any error messages or failed tasks. Ansible provides detailed output that can help you identify what went wrong.
- Verify that you have SSH access to the remote server and that the user specified in the inventory file has the necessary permissions to perform the tasks defined in the playbook.
- Check the variable configurations in the `vars/` directory to ensure that all required variables are set correctly. Missing or incorrect variables can cause the playbook to fail.

In case something is terminally broken and you want to start from scratch, you can use the `cleanup.yaml` playbook to clean up any resources created during the installation process. Please see the [Uninstallation](uninstall.md) page for more details on how to use the cleanup playbook.
