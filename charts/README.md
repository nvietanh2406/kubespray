# I. Install base app
## 1. Longhorn
### Install helm chart
> helm install longhorn longhorn -f longhorn/values.yaml --namespace longhorn-system --create-namespace
### Account
> datxadmin / eddde262-b4a5-4a20-8bdb-3b7f0984c836

## 2. ArgoCD
### Install helm chart
> helm install argocd argocd -f argocd/values.yaml --namespace argocd --create-namespace
### Account
> admin / 6dea16e5-a9e3-4d1e-9d1a-c4a984fea37e

# II. Initialization base app
## 1. ArgoCD
### Login to the ArgoCD server
```
argocd login 10.0.1.55:30080 \
--username admin \
--password 6dea16e5-a9e3-4d1e-9d1a-c4a984fea37e \
--insecure
```
### Create the new repo
```
argocd repo add https://git.datx.com.vn/datx-k8s/k8s-intervpc.git \
--username argocd \
--password "973ecae2-a9c2-4cfe-9e29-730366fd4517" \
--insecure-skip-server-verification
```
### Create the new initialization app

```
argocd app create argocd-apps \
  --name argocd-apps \
  --repo https://git.datx.com.vn/datx-k8s/k8s-intervpc.git \
  --revision master \
  --path helm-charts/argocd-apps \
  --values values.yaml \
  --dest-namespace argocd-apps \
  --dest-server https://kubernetes.default.svc \
  --self-heal \
  --auto-prune \
  --sync-policy auto \
  --upsert
```