# Overview

Welcome to the Polaris documentation overview. This page highlights the key components and tools used in Polaris, as well as the overall architecture of the cluster.

Polaris is built on top of a lightweight Kubernetes distribution called k3s, which provides a simple and efficient way to run Kubernetes clusters. The cluster is managed using Argo CD, a declarative GitOps continuous delivery tool for Kubernetes. Argo CD allows us to define our cluster configuration and applications in Git repositories, making it easy to manage and version control our infrastructure.

Sealed Secrets is used to securely store sensitive information such as passwords, API keys, and other secrets in the cluster. It allows us to encrypt secrets and store them safely in Git repositories without exposing sensitive data.

## Architecture

The cluster uses Argo CD and Kustomize as an `app of apps` to manage the cluster configuration and applications. This means that we have a central repository that defines the overall structure of the cluster, and individual repositories for each application or component. The central repository contains the main configuration files and references to the individual application repositories. Each application repository contains the specific configuration for that application, including any necessary secrets, deployments, and services. This approach allows for a modular and scalable architecture, where we can easily add or remove applications without affecting the overall cluster configuration.

## Tools

TODO

## Monitoring and Logging

TODO
