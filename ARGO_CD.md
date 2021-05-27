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
    -f values-scaleway-testnet.yaml -n argocd .
```

## Access ArgoCD

ArgoCD requires username and password authentication. By default, at the moment of deploying ArgoCD for the first time,
ArgoCD sets the password to the full pod name of the argocd-server. You can easily find it by issueing:

```bash
kubectl get po -n argocd
```

You should see something like:

```bash
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-redis-5cff4b6bf8-qt9j4                    1/1     Running   1          8d
argocd-dex-server-85cbb9b898-pxsh2               1/1     Running   1          8d
argocd-sealed-secrets-5846ff6b9d-4gghf           1/1     Running   1          8d
argocd-repo-server-5f66948994-n2dl7              1/1     Running   1          8d
argocd-server-75df64fb9f-246cn                   1/1     Running   1          8d
argocd-application-controller-7c9d5ffb86-mmw67   1/1     Running   1          8d
```

In this case the password is: `argocd-server-75df64fb9f-246cn`

Alternatively you can set the password at bootstrap time. Steps are:

1. Generate a new password
2. Use the bcrypt to hash the password. You can use this convenience [website](https://www.browserling.com/tools/bcrypt) to do so.
3. Specify the password when bootstrapping the cluster by appending
```bash
--set "argo-cd.configs.secret.argocdServerAdminPassword"='<bcrypt_hashed_password>' \
--set "argo-cd.configs.secret.argocdServerAdminPasswordMtime=$(date +%FT%T%Z)" \
```

## How to deploy a different or test version

> TODO: explain the role of deploying from a branch

## ArgoCD FAQ

The ArgoCD website has a great [FAQ Section](https://argo-cd.readthedocs.io/en/stable/faq/)
