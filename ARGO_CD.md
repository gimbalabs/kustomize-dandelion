# ArgoCD 

## ArgoCD Helm Chart

* github [repo](https://github.com/argoproj/argo-helm)

## Bootstrap

Before bootstrapping ArgoCD, it's necessary to fetch all Helm dependencies.

```shell
kubectl create ns argocd
helm dependency build
helm dependency update

helm upgrade --install argocd \
    --set git.targetRevision=argocd \
    --set clusterName=scaleway -f values-scaleway-testnet.yaml -n argocd .
```

## App of Apps

Bootstrapping a cluster with the AppOfApps approach

https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/
