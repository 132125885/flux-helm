# Project Setup Guide

This guide walks you through setting up a Kubernetes (k8s) cluster with Flux for continuous deployment. Follow these steps carefully to configure your environment, bootstrap Flux in your k8s cluster, and create a WordPress deployment using a Helm chart.

## Setting Up Environment Variables

First, add the necessary environment variables. Replace the values with your actual GitHub organization and token.

```bash
export GITHUB_ORG="Github account"
export GITHUB_TOKEN="GitHub Token"
export GITHUB_PERSONAL=true
export WORDPERSS_HELM_REPO='https://github.com/132125885/flux-helm.git'
export VERSION="19.2.3"
```

## Initial Cluster Setup

Perform this mandatory step for new k8s clusters.

```bash
kubectl create namespace staging
```

## Bootstrap Flux

Create a Flux fleet in GitHub and bootstrap Flux in your k8s cluster with the following command:

```bash
flux bootstrap github \
    --owner $GITHUB_ORG \
    --repository flux-fleet \
    --branch main \
    --path apps \
    --personal $GITHUB_PERSONAL
```

## Verify Flux System Pods

Check if the Flux system pods have been created successfully.

```bash
kubectl --namespace flux-system get pods
```

## Clone the Flux Fleet Repository

Clone the newly created repository to your local machine.

```bash
git clone git@github.com:$GITHUB_ORG/flux-fleet.git
```

## Change Directory

Change into the `flux-fleet` directory.

```bash
cd flux-fleet
```

## Verify the Clone

Check if everything required has been cloned correctly.

```bash
ls -1 apps/flux-system
```

## Create New Source and Kustomization

Create a new source of git type and a kustomization for it.

```bash
flux create source git wordpress-git \
    --url $WORDPERSS_HELM_REPO \
    --branch master \
    --interval 30s \
    --export \
    | tee apps/wordpress-git.yaml
	
flux create kustomization wordpress-git \
    --source wordpress-git \
    --path "./" \
    --prune true \
    --interval 1m \
    --export \
    | tee -a apps/wordpress-git.yaml
```

## Update Staging Configuration

Append the following to `apps/wordpress-git.yaml`.

```bash
cat << EOF  >> ./apps/wordpress-git.yaml
  images:
    - name: ghcr.io/$GITHUB_ORG/wordpress
      newName: ghcr.io/$GITHUB_ORG/wordpress
      newTag: $VERSION
EOF
```

## Create WordPress Deployment

Deploy WordPress from a Helm chart.

```bash
flux create source helm $GITHUB_ORG \
  --url=oci://ghcr.io/$GITHUB_ORG \
  --interval=1m \
  --export | tee apps/helm-source.yaml

flux create helmrelease wordpress \
  --source=HelmRepository/$GITHUB_ORG \
  --chart=wordpress \
  --target-namespace=staging \
  --export | tee apps/wordpress-release.yaml
```

## Commit Changes

Add, commit, and push your changes to the repository.

```bash
git add .
git commit -m "add git source"
git push
```

## Monitor Flux Resources

Use the `watch` command to continuously monitor the status of Flux resources.

```bash
watch flux get sources git
watch flux get kustomizations
watch flux get source helm
watch flux get helmreleases
```

## Helm chart update and new package creation

Use this commands to add all required parameters for Wordpress K8S Helm deployment (sauch ass passwords and etc.)

Clone the repo:

```bash
git clone git@github.com:$GITHUB_ORG/flux-fleet.git
```

Make changes, build package:

```bash
git add .
git commit -m "update version:$VERSION"
git push
echo $GITHUB_TOKEN | helm registry login ghcr.io/$GITHUB_ORG --username $GITHUB_ORG --password-stdin
helm push ./wordpress-$VERSION.tgz oci://ghcr.io/$GITHUB_ORG
```

Other information regarding Wordpress Helm Chart editing could be found in <https://github.com/bitnami/charts/tree/main/bitnami/wordpress/#installing-the-chart>.
