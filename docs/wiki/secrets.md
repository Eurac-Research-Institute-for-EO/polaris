# Secret management

During the first instllation, we have edited the `cluster_secrets.example.yaml` file to set up the secrets for the default services. Make a copy of this file and rename it to `cluster_secrets.yaml`, and fill in the appropriate values for your environment.

These are installed via the `bootstrap` Ansible playbook, which performs the whole installation.

In case we need to only update the secrets in an existing instance, we can use the `secrets.yaml` playbook to only update the existing configuration.

The playbook is located in the `ansible` directory and can be run with the following command:

```bash
ansible-playbook -i inventory secrets.yaml --ask-become-pass
```

The Pods will not automatically update with the new secrets, so we need to restart the Pods to apply the changes. We can do this by deleting the Pods, and they will be recreated with the new secrets.

For example, to restart the Grafana and openEO Pods, we can use the following commands:

**Note:** This requires you to have terminal access to the cluster and the appropriate permissions to manage the deployments.

```bash
kubectl rollout restart deployment/monitoring-grafana -n monitoring"
kubectl rollout restart deployment/openeo-openeo-argo -n openeo"
```
