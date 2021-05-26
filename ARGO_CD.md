# ArgoCD 

In their own words:

> Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

To learn more about ArgoCD visit their [website](https://argo-cd.readthedocs.io/en/stable/)

## ArgoCD Helm Chart

In order to deploy ArgoCD, we will use its helm charts. They are available at the following github 
[repo](https://github.com/argoproj/argo-helm)

## App of Apps

The deployment of ArgoCD and Dandelion, follows the [App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) 
approach.

This means that the ArgoCD will first install itself and then proceed to install all the required components.

## Bootstrap

Before bootstrapping ArgoCD, it's necessary to fetch all Helm dependencies.

```shell
helm dependency build
helm dependency update

kubectl create ns argocd

helm upgrade --install argocd \
    --set git.targetRevision=argocd \
    --set clusterName=scaleway -f values-scaleway-testnet.yaml -n argocd .
```

## How to deploy a different or test version

> TODO: explain the role of deploying from a branch
