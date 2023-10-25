### Install of few packages
```
brew install kind kubectl argoproj/tap/kubectl-argo-rollouts helm k6
```

### Clone the Repo
```
git clone git@github.com:pree-dew/canary-deployment-with-argo-helm.git
```

### Create kind cluster
```
kind create cluster --image kindest/node:v1.23.4 --config=kind/cluster.yaml
```

### Install nginx ingress controller
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx/ingress-nginx --name-template ingress-nginx --create-namespace -n ingress-nginx --values kind/ingress-nginx-values.yaml --version 4.0.19 --wait
```

### Install argo rollout
```
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo/argo-rollouts --name-template argo-rollouts --create-namespace -n argo-rollouts --set dashboard.enabled=true --version 2.14.0 --wait
```

### Create kubectl secrets by using
```
kubectl create secret generic kubepromsecret-write --from-literal=username='xxx' --from-literal=password='xxx' -n prometheus
kubectl create secret generic kubepromsecret-read --from-literal=username='xxx' --from-literal=password='xxx' -n prometheus
```

### Install prometheus
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-community/kube-prometheus-stack --name-template prometheus --create-namespace -n prometheus --version 34.8.0 --wait -f external_tsdb_configuration_values.yaml
```

### Create a values.yaml file to provide additional details to rollout
```
cd sample-app/helm-charts/argo-rollouts/
vim values.yaml

replicas: 1
image:
  registry: ghcr.io
  repository: jhandguy/canary-deployment/sample-app
  tag: 1.0.0
env:
  # -- Last9 api call
  last9_url: "https://gamma.last9.io/api/v4/organizations/last9/domain_events"
  # -- Last9 api token
  last9_token: xxx
notifications:
  # -- Enable notifications controller
  enabled: true
prometheus:
  enabled: false
  # -- read url with username:password format
  address: https://username:password@read-gamma-tsdb.last9.io/hot/v1/metrics/xxx/sender/last9
```

### Automating rollouts
```
helm install sample-app/helm-charts/argo-rollouts --name-template sample-app --create-namespace -n sample-app --set prometheus.enabled=true
```

### Run the argo dashboard
```
kubectl argo rollouts dashboard -n argo-rollouts &
```

### Update the image to shift to canary
```
kubectl argo rollouts set image sample-app sample-app=ghcr.io/jhandguy/canary-deployment/sample-app:latest -n sample-app
```

### Check the anlaysis run by
```
kubectl get analysisrun -n sample-app
```

### Hit canary deployment by
```
curl localhost/success -H "Host: sample.app" -H "X-Canary: always"
```


