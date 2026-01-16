# HyperShift Platform GitOps

This repository contains GitOps manifests for managing a HyperShift-enabled OpenShift platform.

## Structure

```
hypershift-platform-gitops/
├── bootstrap/
│   └── argocd-apps.yaml          # App-of-apps for Argo CD
├── platform/
│   ├── mce/
│   │   ├── namespace.yaml
│   │   ├── operatorgroup.yaml
│   │   ├── subscription.yaml
│   │   └── multiclusterengine.yaml
│   └── clusters-namespace.yaml
└── clusters/
    └── poc-cluster/
        ├── hostedcluster.yaml
        └── nodepool.yaml
```

## Components

### Platform

* **MCE (Multicluster Engine)**: Provides HyperShift capability for hosted clusters
* **Clusters Namespace**: Namespace where hosted clusters are created

### Clusters

* **poc-cluster**: A KubeVirt-based hosted cluster with 2 worker VMs (8 vCPU, 32GB RAM each)

## Prerequisites

* OpenShift 4.18+ with bare metal workers (for KubeVirt)
* OpenShift Virtualization operator installed
* OpenShift GitOps (Argo CD) installed

## Manual Deployment

If not using GitOps, apply manifests in order:

```bash
# 1. Install MCE operator
kubectl apply -f platform/mce/namespace.yaml
kubectl apply -f platform/mce/operatorgroup.yaml
kubectl apply -f platform/mce/subscription.yaml

# 2. Wait for operator to install
kubectl wait --for=condition=CatalogSourcesUnhealthy=False csv -n multicluster-engine --all --timeout=300s

# 3. Create MultiClusterEngine
kubectl apply -f platform/mce/multiclusterengine.yaml

# 4. Wait for HyperShift
kubectl wait --for=condition=Available mce/multiclusterengine --timeout=600s

# 5. Create clusters namespace
kubectl apply -f platform/clusters-namespace.yaml

# 6. Copy pull secret
kubectl get secret pull-secret -n openshift-config -o yaml | \
  sed 's/namespace: openshift-config/namespace: clusters/' | \
  kubectl apply -f -

# 7. Create hosted cluster
kubectl apply -f clusters/poc-cluster/hostedcluster.yaml
kubectl apply -f clusters/poc-cluster/nodepool.yaml
```

## Accessing the Hosted Cluster

```bash
# Get kubeconfig
kubectl get secret -n clusters poc-cluster-admin-kubeconfig \
  -o jsonpath='{.data.kubeconfig}' | base64 -d > /tmp/poc-cluster-kubeconfig

# Use it
export KUBECONFIG=/tmp/poc-cluster-kubeconfig
kubectl get nodes
```
