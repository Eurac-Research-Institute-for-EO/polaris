# Deployment

This section provides instructions for deploying Polaris from scratch. It describes each step of the deployment process, including infrastructure setup, application deployment, and configuration.

## Infrastructure Setup

Obtain a fresh virtual machine. The base cluster itself does not require significant resources but for a smooth experience and some overhead for future applications, we recommend at least 16GB of RAM and 4 vCPUs. (Largely depends on your OS and other applications you want to run on the cluster, but this is a good starting point for a small cluster with some room for growth).

The base installation of Polaris includes a single-node k3s cluster with ArgoCD and Sealed Secrets installed. Later on Argo CD will be used to deploy and manage applications on the cluster, while Sealed Secrets will be used to securely store sensitive information.

## Ansible Setup

The first step is to set up the infrastructure using Ansible. The Ansible playbooks are located in the `ansible` directory of the project. Follow the instructions in the [Ansible Setup](./ansible.md) section to run the playbook and set up the base cluster.

## ArgoCD Deployment

The playbook will deploy ArgoCD on the cluster. The ArgoCD deployment manifest is located in the `argocd` directory. It includes the configuration for GitHub authentication using Dex, as well as the definition of an ArgoCD project and a master application that monitors the Polaris repository.

## Application management

WIP
