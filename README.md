# Requirements

* Kubernetes. For local k8s development environment, we encourage you to use [k3s]/[k3d] lightweight k8s distro. Note you need to use k8s <=v1.21 by the moment. 
* [kubectl]
* [Kustomize] (>=v4.2)
* [helm] (>=3.5.3)
* `bash`. If you are using MacOS to deploy using k3d on docker, you will find problems using `zsh` as shell while copy pasting :)

# Local deployment

There currently are two ways to deploy Dandelion:

* Render and deploy Kustomize manifests
* Leverage ArgoCD to perform more advanced deploys. Check the specific docs [here](/ARGO_CD.md)

## Minimum Requirements

| Network | CPU    | RAM   | DISK  |
| ------- | ------ |------ | ----  |
| Testnet | 4vCPU  | 8GB   | 50gb  |
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

You can also export the `kubeconfig` file from `k3d` like this:
``` 
CLUSTER_NAME=dandelion-dev
k3d kubeconfig merge ${CLUSTER_NAME} --output ~/.kube/dandelion-dev.yaml
```

So in case you have different clusters and don't want to deal with contexts (having a single huge `kubeconfig` file, you can choose which cluster/file to use like this:
```
export KUBECONFIG=$HOME/.kube/dandelion-dev.yaml
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
OVERLAY=iohk-preprod-full
OUTPUT_FILE=overlays/${OVERLAY}/output.yaml # this will contain the whole deploy manifest
kustomize build \
  --enable-helm \
  overlays/${OVERLAY} > overlays/${OVERLAY}/output.yaml
```

## Deploy k8s manifests

You need to execute one of these options from the top dir of this git repository:

* A) Using plain `kubectl`:
```
NAMESPACE=dandelion-iohk-preprod
OVERLAY=iohk-preprod-full
kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}
kubectl apply --validate=false -n ${NAMESPACE} -f overlays/${OVERLAY}/output.yaml
```
* B) Using convenient [kapp] tool to get diff between local manifests and currently deployed ones in cluster:
```
OVERLAY=iohk-preprod-full
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
NETWORK=iohk-preprod
NAMESPACE=dandelion-${NETWORK}
kubectl apply -n ${NAMESPACE} -f base/dandelion-${NETWORK}-public-ingresses/ingress.yaml
```
* Deploy mainnet ingresses:
```greater than 
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
NAMESPACE=dandelion-iohk-preprod
kubectl get ingress -n ${NAMESPACE}
```
* Check postgres[t] db tables
```
ENDPOINT=postgrest-api.iohk-preprod.local
curl -kL -H "Host: ${ENDPOINT}" \
  https://localhost
```
* Check sync progress
```
ENDPOINT=graphql-api.iohk-preprod.local
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
NAMESPACE=dandelion-iohk-preprod
# get read-only user/pass
PGUSER=$(kubectl get secret/init0-postgresql-ha-pgpool-custom-users \
  -n ${NAMESPACE} \
  --template='{{ index .data "usernames" }}' | base64 -d)
PGPASSWORD=$(kubectl get secret/init0-postgresql-ha-pgpool-custom-users \
  -n ${NAMESPACE} \
  --template='{{ index .data "passwords" }}' | base64 -d)

echo "User: ${PGUSER} Pass: ${PGPASSWORD}"
# expose port
kubectl port-forward -n dandelion-iohk-preprod init0-postgresql-ha-postgresql-0 5432:5432
```

## Start from the scratch:

* Destroy namespace
```
NAMESPACE=dandelion-iohk-preprod
kubectl delete ns ${NAMESPACE}
```
* Or even destroy the whole cluster
```
CLUSTER_NAME=dandelion-iohk-preprod
k3d cluster delete  ${CLUSTER_NAME}
```
* Just [deploy again :)](#deploy-k8s-manifests)

## Resync db only

```
NAMESPACE=dandelion-iohk-preprod

kubectl scale statefulset -n ${NAMESPACE} \
  --replicas=0 \
  cardano-db-sync init0-postgresql-ha-postgresql

kubectl delete pvc -n ${NAMESPACE} \
  dbsync-statedir-cardano-db-sync-0 \
  data-init0-postgresql-ha-postgresql-0

kubectl delete job --all -n ${NAMESPACE}

kubectl scale statefulset -n ${NAMESPACE} \
  --replicas=1 \
  cardano-db-sync init0-postgresql-ha-postgresql

for jobfile in base/cardano-db-sync/job-ro-user-create.yaml base/koios/job-deploy-koios-sql.yaml
do
  kubectl apply -n ${NAMESPACE} -f ${jobfile}
done

kubectl rollout restart deploy -n ${NAMESPACE} postgrest koios-api 
```
 

[docker]: https://docs.docker.com/engine/install/
[kustomize]: https://kustomize.io
[kubectl]: https://kubernetes.io/docs/tasks/tools/#kubectl
[k3d]: https://k3d.io
[k3s]: https://k3s.io
[kapp]: https://github.com/vmware-tanzu/carvel-kapp
[traefik]: https://traefik.io/traefik/
[helm]: https://helm.sh/docs/intro/install/
