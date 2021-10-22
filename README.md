# Requirements

* Kubernetes. For local k8s development environment, we encourage you to use [k3s]/[k3d] lightweight k8s distro. Note you need to use k8s <=v1.21 by the moment. 
* [kubectl]
* [Kustomize] (greater than v4.1)
* [helm]
* `bash`. If you are using MacOS to deploy using k3d on docker, you will find problems using `zsh` as shell while copy pasting :)

# Local deployment

There currently are two ways to deploy Dandelion:

* Render and deploy Kustomize manifests
* Leverage ArgoCD to perform more advanced deploys. Check the specific docs [here](/ARGO_CD.md)

## Minimum Requirements

| Network | CPU    | RAM   | DISK  |
| ------- | ------ |------ | ----  |
| Testnet | 4vCPU  | 8GB   | 30gb  |
| Mainnet | 24vCPU | 24GB  | 120gb |

NOTE: vCPUs are recommended to be dedicated for the db synchronization to stay close to blokchain tip.

## Setup local k3d 

Assuming you are already running [docker] locally, and have both, [kubectl] and [k3d] binaries installed, you could create a `k3d` cluster like this (customize port mappings if you feel to):
```
CLUSTER_NAME=dandelion-dev
k3d cluster create \
  -p "80:80@loadbalancer" \
  -p "443:443@loadbalancer" \
  ${CLUSTER_NAME}
kubectl config use-context k3d-${CLUSTER_NAME}
kubectl get pods -A
```

## Clone this repository

* For example, using command line like this:
```
mkdir Projects
cd Projects
git clone https://gitlab.com/gimbalabs/dandelion/kustomize-dandelion
```
* Then enter the directory and execute the following steps from there:
```
cd kustomize-dandelion
```

## Render k8s manifests

Before directly rendering an overlay, you might want to enable/disable some components (`rosetta`, `postgrest`...) by removing the corresponding `bases` from the `kustomization.yaml` in the overlay dir.

You need to execute this from the top dir of this git repository:
 
``` 
OVERLAY=testnet-full
OUTPUT_FILE=overlays/${OVERLAY}/output.yaml # this will contain the whole deploy manifest
kustomize build \
  --enable-helm \
  overlays/${OVERLAY} > overlays/${OVERLAY}/output.yaml
```

## Deploy k8s manifests

You need to execute one of these options from the top dir of this git repository:

* A) Using plain `kubectl`:
```
NAMESPACE=dandelion-testnet
OVERLAY=testnet-full
kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}
kubectl apply --validate=false -n ${NAMESPACE} -f overlays/${OVERLAY}/output.yaml
```
* B) Using convenient [kapp] tool to get diff between local manifests and currently deployed ones in cluster:
```
OVERLAY=testnet-full
NAMESPACE=dandelion-${OVERLAY}
APP_NAME=${NAMESPACE}
kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}
kapp deploy -c \
  --app ${APP_NAME} \
  --namespace ${NAMESPACE} \
  --file overlays/${OVERLAY}/output.yaml
```

## Enable your deployment to be exposed behind the community service

Execute this from the top dir of this git repository:

* Deploy testnet ingresses:
```
NETWORK=testnet
NAMESPACE=dandelion-${NETWORK}
kubectl apply -n ${NAMESPACE} -f base/dandelion-${NETWORK}-public-ingresses/ingress.yaml
```
* Deploy mainnet ingresses:
```
NETWORK=mainnet
NAMESPACE=dandelion-${NETWORK}
kubectl apply -n ${NAMESPACE} -f base/dandelion-${NETWORK}-public-ingresses/ingress.yaml
```

## Deploy in multi-node

In order to deploy services into different kubernetes nodes/workers you can use the affinity-ready overlay `${NETWORK}-multi-node` (check patches [here](/base/mainnet-affinity-patches)). We've coupled some services (ie, db ones), decoupled some others (node and graphql) and let others to float around the cluster (any other "consumer" API). 
Besides the services label below, you'll also need to add the corresponding network label to your workers: `k8s.dandelion.link/cardano-network=mainnet`.


| Role            | Services                          | Label                                   | Recommended Node Size (mainnet) | Recommended Node Size (testnet) |
| --------------- | --------------------------------- |---------------------------------------- | ------------------------------- | ------------------------------- |
| db-main         | postgres, pgpool, cardano-db-sync | k8s.dandelion.link/role=db-main         | 4vCPU/16gb RAM                  | 2vCPU/2gb RAM                   |
| cardano-graphql | cardano-graphql                   | k8s.dandelion.link/role=cardano-graphql | 4vCPU/8gb RAM                   | 2vCPU/2gb RAM                   |
| cardano-node    | cardano-node                      | k8s.dandelion.link/role=cardano-node    | 4vCPU/8gb RAM                   | 2vCPU/2gb RAM                   |
| observability   | prometheus, grafana, loki         | k8s.dandelion.link/role=observability   | 2x 2vCPU/2gb RAM                | 2vCPU/2gb RAM                   |


## Usage

Given that everything is listening on `localhost` we need to specify `Host` headers for the requests to be properly routed by the `Ingress` (on k3d, [traefik] by default).
If you prefer, you could fake these hostnames to point to `127.0.0.1` on your `/etc/hosts` (linux/darwin) file so you don't need to specify them for each request.

* Get available endpoints:
```
NAMESPACE=dandelion-testnet
kubectl get ingress -n ${NAMESPACE}
```
* Check postgres[t] db tables
```
ENDPOINT=postgrest-api.testnet.local
curl -kL -H "Host: ${ENDPOINT}" \
  https://localhost
```
* Check sync progress
```
ENDPOINT=graphql-api.testnet.local
curl -kL \
     -H "Host: ${ENDPOINT}" \
     -H 'Accept-Encoding: gzip, deflate, br' \
     -H 'Content-Type: application/json' \
     -H 'Accept: application/json' \
     --data-binary '{"query":"query cardanoDbSyncProgress {\n    cardanoDbMeta {\n        initialized\n        syncPercentage\n    }\n}\n"}' \
     'https://localhost/'
```

# Operations

## External database access

If you want to explore the db from outside the cluster (GUI tools, et al), you can expose postgres port locally:
```
NAMESPACE=dandelion-testnet
# get read-only user/pass
PGPUSER=$(kubectl get secret/init0-postgresql-ha-pgpool-custom-users \
  -n ${NAMESPACE} \
  --template='{{ index .data "usernames" }}' | base64 -d)
PGPASSWORD=$(kubectl get secret/init0-postgresql-ha-pgpool-custom-users \
  -n ${NAMESPACE} \
  --template='{{ index .data "passwords" }}' | base64 -d)

echo "User: ${PGUSER} Pass: ${PGPASSWORD}"
# expose port
kubectl port-forward -n dandelion-testnet init0-postgresql-ha-postgresql-0 5432:5432
```

## Start from the scratch:

* Destroy namespace
```
NAMESPACE=dandelion-testnet
kubectl delete ns ${NAMESPACE}
```
* Or even destroy the whole cluster
```
CLUSTER_NAME=dandelion-testnet
k3d cluster delete ${CLUSTER_NAME}
```
* Just [deploy again :)](#deploy-k8s-manifests)

[docker]: https://docs.docker.com/engine/install/
[kustomize]: https://kustomize.io
[kubectl]: https://kubernetes.io/docs/tasks/tools/#kubectl
[k3d]: https://k3d.io
[k3s]: https://k3s.io
[kapp]: https://github.com/vmware-tanzu/carvel-kapp
[traefik]: https://traefik.io/traefik/
[helm]: https://helm.sh/docs/intro/install/
