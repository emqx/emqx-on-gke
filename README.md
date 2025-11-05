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
export PROJECT=<your project id>
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
kubectl create namespace emqx
./install-emqx-operator.sh
kubectl apply -f emqx.yaml
kubectl -n emqx wait --for=condition=Ready emqx emqx --timeout=120s
kubectl -n emqx get svc
```

Retrieve IP address of EMQX Dashboard:

```bash
kubectl -n emqx get svc emqx-dashboard -o json | jq '.status.loadBalancer.ingress[0].ip'
```

Now you can open this IP address in the browser by visiting `http://<DASHBOARD_IP>:18083` (default credentials: `admin`/`public`).

## Operational check

```bash
DASHBOARD_URL=$(kubectl -n emqx get svc emqx-dashboard -o json | jq '.status.loadBalancer.ingress[0].ip' -r)
curl -u admin:public "http://$DASHBOARD_URL:18083/api/v5/status"
Node emqx@emqx-core-<***>-0.emqx-headless.emqx.svc.cluster.local is started
emqx is running
```

## Deploy test client

```bash
kubectl apply -f mqttx.yaml
kubectl -n emqx logs -f mqttx-<***>-<***> mqttx-cli
```

## Add ACLs via config change

First, create a secret from acl.txt, and make sure it propagated to all nodes:

```bash
kubectl -n emqx create secret generic emqx-acl --from-file=./acl.txt
kubectl -n emqx apply -f emqx-acl-1.yaml
kubectl -n emqx wait --for=condition=Ready emqx emqx --timeout=120s
```

Then, apply config change:

```bash
kubectl -n emqx apply -f emqx-acl-2.yaml
```

Verify:

```bash
core0=$(kubectl -n emqx get pods -l 'apps.emqx.io/instance=emqx,apps.emqx.io/db-role=core' -o json | jq --raw-output '.items[0].metadata.name')
kubectl -n emqx exec -it $core0 -c emqx -- emqx ctl conf show authorization
authorization {
  cache {
    enable = true
    excludes = []
    max_size = 32
    ttl = "1m"
  }
  deny_action = ignore
  no_match = deny
  node_cache {
    cache_ttl = "1m"
    enable = false
    max_count = 1000000
    max_memory = "100MB"
  }
  sources = [
    {
      enable = true
      path = "/mnt/acl/acl.txt"
      type = file
    }
  ]
}
```

## Enable data persistence

```bash
kubectl apply -f emqx-persistent-volumes.yaml
kubectl -n emqx wait --for=condition=Ready emqx emqx --timeout=120s
core0="$(kubectl get pods -l 'apps.emqx.io/instance=emqx,apps.emqx.io/db-role=core' -o json | jq --raw-output '.items[0].metadata.name')"
kubectl -n emqx exec -it ${core0} -- emqx ctl ds info
```

It may happen that old core nodes are still in the cluster after enabling data persistence. In this case, you need to manually kick them out of the cluster (mind the different IDs):

```bash
kubectl -n emqx exec -it ${core0} -- emqx ctl cluster force-leave emqx@emqx-core-<***>-0.emqx-headless.emqx.svc.cluster.local
kubectl -n emqx exec -it ${core0} -- emqx ctl cluster force-leave emqx@emqx-core-<***>-1.emqx-headless.emqx.svc.cluster.local
```

## Rebalance after change in cluster topology

```bash
kubectl apply -f rebalance.yaml
```

## Cleanup

```bash
kubectl delete -f mqttx.yaml
kubectl delete -f emqx.yaml
gcloud container clusters delete emqx
```

## Troubleshoting

```bash
# watch events
kubectl get events --all-namespaces --watch
# ssh on the GKE pool node, e.g.
gcloud compute ssh gke-emqx-default-pool-<***>-bjcg
# list pods
crictl pods
# list containers
crictl ps
# get normal shell (then you can apt-get update, etc)
/usr/bin/toolbox
```
