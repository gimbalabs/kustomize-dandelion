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

> **NOTE**: Please check section below on how to set a custom password to access ArgoCD. 

```shell
cd argocd-bootstrap

helm dependency build
helm dependency update

helm upgrade \
    --create-namespace \
    --namespace argocd \
    --install argocd \
    -f values-scaleway-testnet.yaml \
    .
```

## Deploy more apps

```
APP_PROVIDER=k3s
APP_NETWORK=testnet
APP_SUFFIX=-v9-0-0
APP_NAME=dandelion-${APP_NETWORK}${APP_SUFFIX}
APP_REVISION=refactor/add-base-and-multi-node-overlays
argocd app create \
  ${APP_NAME} \
  --repo https://gitlab.com/gimbalabs/dandelion/kustomize-dandelion.git \
  --revision ${APP_REVISION} \
  --path applications/main-app \
  --helm-set git.targetRevision=${APP_REVISION} \
  --values-literal-file values-${APP_PROVIDER}-${APP_NETWORK}.yaml \
  --dest-server https://kubernetes.default.svc
```

## Access ArgoCD

ArgoCD requires username and password authentication. By default, at the moment of deploying ArgoCD for the first time, username is `admin` and password is set to the full pod name of the `argocd-server`.
You can easily find the password and access the service by following these steps:

* Get default username/password:
```bash
POD_NAME=$(kubectl get pod -n argocd --selector 'app.kubernetes.io/name=argocd-server' --template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo -e "Username: admin\nPassword: ${POD_NAME}"
```
* Expose ArgoCD server port locally on your machine so you can access it via http://localhost:8080:
```bash
kubectl port-forward -n argocd ${POD_NAME} 8080:8080
```

Alternatively you can set the password at bootstrap time. Steps are:

1. Generate a new password
2. Use the bcrypt to hash the password. You can use this convenience [website](https://www.browserling.com/tools/bcrypt) to do so.
3. Specify the password when bootstrapping the cluster by appending
```bash
helm upgrade --install argocd \
    --set git.targetRevision=DND-Observability-Take1 \
    --set "argo-cd.configs.secret.argocdServerAdminPassword"='$2a$10$GG4A3RZ.TNYNGoVoPmlCrOO9PgwVy9lTN3s.mhfLO1JwzCALpuoLW' \
    --set "argo-cd.configs.secret.argocdServerAdminPasswordMtime=$(date +%FT%T%Z)" \
    -f values-k3s-testnet.yaml -n argocd .
```

## How to deploy a different or test version

> TODO: explain the role of deploying from a branch

## ArgoCD FAQ

The ArgoCD website has a great [FAQ Section](https://argo-cd.readthedocs.io/en/stable/faq/)
