# ArgoCD 

## ArgoCD Helm Chart

* github [repo](https://github.com/argoproj/argo-helm)

## Bootstrap

Before bootstrapping ArgoCD, it's necessary to fetch all Helm dependencies.

```shell
kubectl create ns argocd
helm dependency build
```

## App of Apps
