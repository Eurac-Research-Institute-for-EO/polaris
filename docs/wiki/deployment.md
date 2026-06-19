# Deployment

Polaris can be deployed in two ways. Both use [Ansible](https://docs.ansible.com/) to go from a bare server (or servers) to a running [K3s](https://k3s.io/) cluster with ArgoCD and the Polaris application set.

## Choose your setup

### [Single Node](single-node.md)

The full Polaris cluster on **one server** — control plane and workloads on a single K3s node. This is the original setup, driven from the `ansible/` directory. Start here for a complete, self-contained cluster.

### [Multinode](multinode/multinode.md)

A **3-node cluster** — one master and two workers — driven from the `multinode/` directory. A simpler, foundational variant that adds node provisioning and cluster-wide DNS configuration on top of the same ArgoCD core.

## What they share

Both setups stand on the same ArgoCD foundation: Keycloak OIDC single sign-on, the `polaris` AppProject, and the app-of-apps Application that pulls in the rest of the workloads. The difference is the number of nodes and how secrets are applied — see each section for the details.
