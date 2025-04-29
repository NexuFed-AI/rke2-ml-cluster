# Deploy fleet and configuration to cluster
Fleet is a GitOps tool for Kubernetes. It is used to deploy and manage applications on Kubernetes clusters.

## Install Fleet
```bash
PASSWORD=${PASSWORD:-"admin"}

BASE_DOMAINE=${BASE_DOMAINE:-"local"}
CONTROL_DNS="ml-cluster.${BASE_DOMAINE}" # This is the DNS name of the master. It is used for accessing the Rancher UI.
RANCHER_PASSWORD=${RANCHER_PASSWORD:-$PASSWORD} # This is the password for the Rancher admin user.

JETSTACK_VERSION=${JETSTACK_VERSION:-$(curl -L --silent "https://api.github.com/repos/jetstack/cert-manager/releases/latest" | jq -r .tag_name)} # This is the version of cert-manager to install.
# CONTROL_IP=${CONTROL_IP:-$(hostname -I | cut -d' ' -f2)} # This is the virtual IP of the cluster.
# CONTROL_IP=${CONTROL_IP:-$(hostname -I | cut -d' ' -f1)} # This is the IP address of the master. It is used for accessing the Rancher UI.
# CONTROL_DNS=${CONTROL_DNS:-"rancher.$CONTROL_IP.nip.io"} # This is the DNS name of the master. It is used for accessing the Rancher UI, if no DNS name is provided.
# HOST_IP=${HOST_IP:-$(hostname -I | cut -d' ' -f1)}


# Wait for ingress to be ready
while ! kubectl rollout status daemonset -n kube-system rke2-ingress-nginx-controller --timeout=60s; do sleep 2 ; done

# add helm
curl -#L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# add needed helm charts
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add jetstack https://charts.jetstack.io

kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/$JETSTACK_VERSION/cert-manager.crds.yaml
helm upgrade -i cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace

kubectl rollout status deployment -n cert-manager cert-manager --timeout=60s
kubectl rollout status deployment -n cert-manager cert-manager-webhook --timeout=60s

# Install Rancher
helm upgrade -i rancher rancher-latest/rancher \
    --create-namespace \
    --namespace cattle-system \
    --set hostname=$CONTROL_DNS \
    --set bootstrapPassword=$PASSWORD \
    --set replicas=1
    
    # --set ingress.tls.source=letsEncrypt \
    # --set letsEncrypt.email=<your-email-address>
```

## Deploy Fleet Repo
```bash
#!/bin/bash
user=anonymous
read -p "Enter Personal Access Token: " pat
kubectl create secret generic basic-auth-secret -n fleet-local --type=kubernetes.io/basic-auth --from-literal=username=$user --from-literal=password=$pat
kubectl -n fleet-local get fleet
kubectl apply -f fleet-repo.yaml
kubectl -n fleet-local get fleet
```