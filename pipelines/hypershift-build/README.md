# HyperShift Build Pipeline

This directory contains Tekton resources for building the custom HyperShift operator image.

## Prerequisites

1. Create the build namespace:
   ```bash
   kubectl create ns hypershift-build
   ```

2. Create the quay.io credentials secret:
   ```bash
   kubectl create secret generic quay-creds \
     --from-file=.dockerconfigjson=$HOME/.docker/config.json \
     --type=kubernetes.io/dockerconfigjson \
     -n hypershift-build
   ```

## Building the Image

1. Update `pipelinerun.yaml` with your desired:
   * `IMAGE` - Target image (e.g., `quay.io/klape/hypershift:arm64-kubevirt-dns-fix`)
   * `GIT_REVISION` - Branch or tag to build (e.g., `arm64-kubevirt-support`)

2. Run the pipeline:
   ```bash
   kubectl create -f pipelinerun.yaml
   ```

3. Monitor progress:
   ```bash
   kubectl get pipelinerun -n hypershift-build -w
   kubectl logs -n hypershift-build -l tekton.dev/pipelineRun -f
   ```

## Source Repository

The pipeline clones from `https://github.com/kylape/hypershift.git` and uses `Dockerfile.tekton` which is designed for Tekton builds with public base images.

## Notes

* The build uses `golang:1.24` as the builder image (public Docker Hub image)
* Final image is based on `registry.access.redhat.com/ubi9:latest`
* Build includes all HyperShift components: hypershift-operator, control-plane-operator, control-plane-pki-operator, karpenter-operator, CLI tools
