# VM Information
| K8S Role    | IP Addres     | Hostname              | vCPU    | RAM       | Disk  |
|-------------| --------------|-----------------------|---------|-----------|-------|
| Master      | 10.0.1.131    | k8s-dev02-master01    | 4vCPU   | 8G RAM    | 100G  |
| Master      | 10.0.1.132    | k8s-dev02-master02    | 4vCPU   | 8G RAM    | 100G  |
| Master      | 10.0.1.133    | k8s-dev02-master03    | 4vCPU   | 8G RAM    | 100G  |
| Worker      | 10.0.1.141    | k8s-dev02-worker01    | 16vCPU  | 48G RAM   | 300G  |
| Worker      | 10.0.1.142    | k8s-dev02-worker02    | 16vCPU  | 48G RAM   | 300G  |
| Worker      | 10.0.1.143    | k8s-dev02-worker03    | 16vCPU  | 48G RAM   | 300G  |
| Worker      | 10.0.1.144    | k8s-dev02-worker04    | 16vCPU  | 48G RAM   | 300G  |
| Worker      | 10.0.1.145    | k8s-dev02-worker05    | 16vCPU  | 48G RAM   | 300G  |

# I. Prerequisites
## 1. Install kubectl
```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client --output=yaml
```

## 2. Enable shell autocompletion
```shell
echo 'source /usr/share/bash-completion/bash_completion' >>~/.bashrc
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
k version --client --output=yaml
```

## 3. Install Helm
```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
ln -s /usr/local/bin/helm /bin/helm
```
## 4. Install ArgoCD CLI
```shell
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

# II. Install Kubespray
***Make sure that all K8s root's home nodes have Ansible's public ssh key added.***
## 1. Clone DatX kubespray repo
```shell
mkdir -p /opt/kubernetes && cd /opt/kubernetes/ 
git clone https://git.datx.com.vn/dso/kubespray.git -b cmc-k8s-dev02
cd kubespray
pip install -r requirements.txt
```
## 2. Setup nodes requestment
```shell
/usr/local/bin/ansible-playbook \
    -i inventory/cmc-k8s-dev02/hosts.yaml \
    --become \
    --become-user=root \
    install-node-requestment.yml
```
## 3. Deploy kubespray with ansible Playbook
```shell
/usr/local/bin/ansible-playbook \
    -i inventory/cmc-k8s-dev02/hosts.yaml \
    --become \
    --become-user=root \
    cluster.yml
```
## 4. Copy K8S Cluster's config file
```shell
mkdir /root/.kube -p
rsync root@10.0.1.131:/etc/kubernetes/admin.conf /root/.kube/config
sed -i 's/127.0.0.1:6443/10.0.1.131:6443/g' /root/.kube/config
```
## 5. Verify K8S cluster status:
```shell
kubectl get node -o wide
kubectl get --raw='/readyz?verbose'
```
## 6. Verify DNS Working:
```shell
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
kubectl get pods dnsutils
sleep 15
kubectl exec -i -t dnsutils -- nslookup kubernetes.default
```

# III. Install base app
## 1. Ingress Nginx
### Install helm chart
```shell
helm upgrade --install ingress-nginx charts/ingress-nginx \
  -f charts/ingress-nginx/values.yaml \
  --namespace ingress-nginx \
  --create-namespace
```
Verify:
```shell
kubectl get po -n ingress-nginx
```

## 2. Longhorn
### Install helm chart
```shell
helm upgrade --install longhorn charts/longhorn \
  -f charts/longhorn/values.yaml \
  --namespace longhorn-system \
  --create-namespace
```
Verify:
```shell
kubectl get po -n longhorn-system
```

### Account
```
datxadmin / eddde262-b4a5-4a20-8bdb-3b7f0984c836
```
## 3. ArgoCD
### Install helm chart
```shell
helm upgrade --install argocd charts/argocd \
  -f charts/argocd/values.yaml \
  --namespace argocd \
  --create-namespace
```
Verify:
```shell
kubectl get po -n argocd
```
### Account
```
admin / 6dea16e5-a9e3-4d1e-9d1a-c4a984fea37e
```
# IV. Initialization base app
## 1. ArgoCD
### Login to the ArgoCD server
```shell
argocd login 10.0.1.142:30080 \
  --username admin \
  --password 6dea16e5-a9e3-4d1e-9d1a-c4a984fea37e \
  --insecure
```
### Create the new repo
```shell
argocd repo add https://git.datx.com.vn/datx-k8s/k8s-intervpc.git \
  --username argocd \
  --password "973ecae2-a9c2-4cfe-9e29-730366fd4517" \
  --insecure-skip-server-verification
```
### Create the new initialization app

```shell
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

# V. Add/Remove a node
Ref:
[Link](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/nodes.md)

# VI. Cleanup
Reset k8s cluster:
```shell
/usr/local/bin/ansible-playbook \
    -i inventory/cmc-k8s-dev02/hosts.yaml \
    --become \
    --become-user=root \
    reset.yml
```
Delete Longhorn's data directory:
```shell
/usr/local/bin/ansible-playbook \
    -i inventory/cmc-k8s-dev02/hosts.yaml \
    --become \
    --become-user=root \
    clean-longhorn-data.yml
```