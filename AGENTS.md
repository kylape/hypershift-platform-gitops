# Hypershift Platform Gitops

* New clusters need the cluster pull secret copied to the hypershift namespace to be able to pull images
* Make all changes to the cluster via this gitops repo and ArgoCD
* Don't use the argocd cli. Just use kubectl commands.
