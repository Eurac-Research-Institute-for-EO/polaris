# Uninstall

Sometimes you may want to uninstall Polaris and all of its components.

To do this, run the `cleanup.yaml` Ansible playbook:

```bash
ansible-playbook -i inventory.ini cleanup.yaml --ask-become-pass
```

This playbook will:

1. Remove all ArgoCD applications, ApplicationSets, and AppProjects (including finalizers to ensure deletion)
2. Delete ArgoCD cluster-scoped resources (ClusterRoles and ClusterRoleBindings)
3. Delete the `argocd` namespace and all ArgoCD resources within it
4. Wait for all ArgoCD resources to be fully deleted before finishing

After running this playbook, Polaris and all of its components should be fully uninstalled from your cluster.

Then you can safely follow the installation instructions again if you want to reinstall Polaris from scratch.
