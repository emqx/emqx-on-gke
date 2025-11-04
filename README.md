# Deploying EMQX on GKE

## Pre-requisites

- [GCP project](https://cloud.google.com/resource-manager/docs/creating-managing-projects)
- [gcloud CLI](https://cloud.google.com/sdk/docs/install)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Run once

```bash
gcloud init
gcloud auth login
gcloud components update
export PROJECT=cs-poc-yfdcyvmaxqv0fixentos0bz
export REGION=europe-west1
export ZONE=europe-west1-b
gcloud auth application-default set-quota-project $PROJECT
gcloud config set project $PROJECT
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
```

## Create EMQX cluster in GKE

```bash
gcloud container clusters create emqx
gcloud container clusters get-credentials emqx
```

## Deploy EMQX via EMQX Operator

```bash
./install-emqx-operator.sh
kubectl create namespace emqx
kubectl apply -f emqx.yaml
kubectl -n emqx wait --for=condition=Ready emqx emqx --timeout=120s
kubectl -n emqx get svc
```

## Rebalance after change in cluster topology

```bash
kubectl apply -f rebalance.yaml
```

## Cleanup

```bash
kubectl delete -f emqx.yaml
gcloud container clusters delete emqx
```

## Troubleshoting

```bash
# watch events
kubectl get events --all-namespaces --watch
# ssh on the GKE pool node, e.g.
gcloud compute ssh gke-emqx-default-pool-8ffb4312-bjcg
# list pods
crictl pods
# list containers
crictl ps
# get normal shell (then you can apt-get update, etc)
/usr/bin/toolbox
```
