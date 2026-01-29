# Hypershift Platform Gitops

* New clusters need the cluster pull secret copied to the hypershift namespace to be able to pull images
* Make all changes to the cluster via this gitops repo and ArgoCD
* Don't use the argocd cli. Use kubectl to manipulate ArgoCD resources to trigger syncs.
* To sync an ArgoCD Application: `kubectl patch application <app-name> -n openshift-gitops --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'`
