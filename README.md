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
gcloud config set project <your project id>
gcloud config set compute/region <your region, e.g. europe-west1>
gcloud config set compute/zone <your zone, e.g. europe-west1-b>
```

## Create EMQX cluster in GKE

```bash
gcloud container clusters create emqx
gcloud container clusters get-credentials emqx
```

## Deploy EMQX via EMQX Operator

Install EMQX Operator
```bash
kubectl create namespace emqx
./install-emqx-operator.sh
```

Install EMQX via Helm Chart

```bash
helm upgrade --install emqx charts/emqx-cr --namespace emqx --create-namespace
```

**OR** install EMQX via `kubectl apply`

```bash
kubectl apply -f emqx.yaml
```

Wait for EMQX to be ready and get services:
```bash
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

It will automatically start sending traffic to EMQX.

## Test EMQX updates via Helm

To reproduce the customer scenario where Helm is used instead of `kubectl apply`, a minimal chart lives in `charts/emqx-cr`. The chart templates the same EMQX custom resource (`fullnameOverride` defaults to `emqx`) so Helm upgrades should trigger the operator.

Initial install (or takeover of an existing `emqx` resource):

```bash
helm upgrade --install emqx charts/emqx-cr \
  --namespace emqx \
  --create-namespace
```

You can now drive updates through Helm by changing values. For quick experiments you can use `--set`, e.g.:

```bash
# Flip the console log level to debug
helm upgrade emqx charts/emqx-cr \
  --namespace emqx \
  --set config.logConsoleLevel=debug
```

For more complex changes (config snippets, listener annotations, storage sizes, replica counts, etc.) drop a values file:

```bash
cat <<'EOF' > /tmp/emqx-values.yaml
config:
  logConsoleLevel: warning
extraConfig: |
  dashboard.listeners.http.bind = 18083
core:
  replicas: 3
EOF

helm upgrade emqx charts/emqx-cr \
  --namespace emqx \
  -f /tmp/emqx-values.yaml
```

This allows you to observe whether the operator reconciles the changes when they originate from Helm. Uninstall when you are done:

```bash
helm uninstall emqx -n emqx
```

## Add ACLs via config change

First, create a secret from acl.txt, and make sure it propagated to all nodes:

```bash
kubectl -n emqx create secret generic emqx-acl --from-file=./acl.txt
kubectl -n emqx apply -f emqx-acl-1.yaml
kubectl -n emqx wait --for=condition=Ready emqx emqx --timeout=10m
```

Then, apply the config change:

```bash
kubectl -n emqx apply -f emqx-acl-2.yaml
```

Check emqx-operator logs:

```bash
kubectl -n emqx logs -l app.kubernetes.io/name=emqx-operator -f
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

## Rebalance after change in cluster topology

Note, it may report `nothing_to_balance`:

```bash
core0=$(kubectl -n emqx get pods -l 'apps.emqx.io/instance=emqx,apps.emqx.io/db-role=core' -o json | jq --raw-output '.items[0].metadata.name')
kubectl -n emqx exec -it $core0 -- emqx ctl rebalance start
kubectl -n emqx exec -it $core0 -- emqx ctl rebalance status
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
