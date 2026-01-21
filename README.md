# HyperShift Platform GitOps

This repository contains GitOps manifests for managing a standalone HyperShift deployment on OpenShift.

## Structure

```
hypershift-platform-gitops/
├── bootstrap/
│   └── argocd-apps.yaml          # App-of-apps for Argo CD
├── platform/
│   ├── hypershift/
│   │   ├── crds.yaml             # HyperShift CRDs
│   │   └── operator.yaml         # HyperShift operator deployment
│   ├── mce/
│   │   ├── namespace.yaml
│   │   ├── operatorgroup.yaml
│   │   ├── subscription.yaml
│   │   └── multiclusterengine.yaml
│   ├── openshift-virtualization/
│   │   ├── namespace.yaml        # openshift-cnv namespace
│   │   ├── operatorgroup.yaml
│   │   ├── subscription.yaml     # CNV operator subscription
│   │   └── hyperconverged.yaml   # HyperConverged CR with VSOCK enabled
│   ├── stackrox/
│   │   ├── 00-namespace.yaml
│   │   ├── 01-secrets.yaml       # TLS certs, passwords, init bundles
│   │   ├── 02-configmaps.yaml
│   │   ├── 03-serviceaccounts.yaml
│   │   ├── 04-services.yaml
│   │   ├── 05-deployments.yaml   # Central, Scanner V4, Sensor, etc.
│   │   ├── 06-daemonsets.yaml    # Collector
│   │   ├── 07-roles.yaml
│   │   ├── 08-rolebindings.yaml
│   │   ├── 09-clusterroles.yaml
│   │   ├── 10-clusterrolebindings.yaml
│   │   ├── 11-pvcs.yaml
│   │   ├── 12-routes.yaml
│   │   └── 13-crds.yaml          # SecurityPolicy CRD
│   └── clusters-namespace.yaml
└── clusters/
    └── poc-cluster/
        ├── hostedcluster.yaml
        └── nodepool.yaml
```

## Components

### Platform

* **HyperShift Operator**: Standalone HyperShift deployment using custom image `quay.io/klape/hypershift:arm64-kubevirt`
* **Multicluster Engine (MCE)**: MCE operator for cluster management
* **OpenShift Virtualization**: KubeVirt-based virtualization platform
  * Includes HyperConverged CR with VSOCK feature gate enabled for VM scanning
* **StackRox (ACS)**: Advanced Cluster Security deployment
  * Central, Scanner V4, Sensor, Admission Control, Collector
  * VM scanning support enabled via `ROX_VIRTUAL_MACHINES=true`
* **Clusters Namespace**: Namespace where hosted clusters are created

### Clusters

* **poc-cluster**: A KubeVirt-based hosted cluster with 2 worker VMs (8 vCPU, 32GB RAM each)

## Prerequisites

* OpenShift 4.18+ with bare metal or virtualization-capable workers
* OpenShift GitOps (Argo CD) installed

## GitOps Deployment

```bash
# 1. Install OpenShift GitOps operator
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# 2. Wait for GitOps operator
oc wait --for=condition=Available deployment/openshift-gitops-server -n openshift-gitops --timeout=300s

# 3. Apply the bootstrap
oc apply -f https://raw.githubusercontent.com/kylape/hypershift-platform-gitops/main/bootstrap/argocd-apps.yaml
```

## Manual Deployment

If not using GitOps, apply manifests in order:

```bash
# 1. Install HyperShift CRDs
kubectl apply -f platform/hypershift/crds.yaml

# 2. Wait for CRDs to be established
kubectl wait --for=condition=Established crd/hostedclusters.hypershift.openshift.io --timeout=60s

# 3. Install HyperShift operator
kubectl apply -f platform/hypershift/operator.yaml

# 4. Wait for operator
kubectl wait --for=condition=Available deployment/operator -n hypershift --timeout=300s

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

## Updating the HyperShift Image

To update the HyperShift operator image:

1. Build and push your new image to `quay.io/klape/hypershift:<tag>`
2. Regenerate the operator manifest:
   ```bash
   cd ~/workspace/src/hypershift
   ./bin/hypershift install render \
     --hypershift-image="quay.io/klape/hypershift:<new-tag>" \
     --limit-crd-install=KubeVirt \
     --outputs=resources \
     --output-file=~/workspace/src/hypershift-platform-gitops/platform/hypershift/operator.yaml
   ```
3. Commit and push to trigger ArgoCD sync
