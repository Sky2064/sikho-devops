# Deploy Wanderblog to AKS

This guide shows minimal steps to run the existing manifests on Azure Kubernetes Service (AKS).

1. Login to Azure and set variables

```powershell
az login
az account set --subscription <YOUR_SUBSCRIPTION_ID>
RESOURCE_GROUP=myResourceGroup
LOCATION=eastus
ACR_NAME=myWanderblogAcr
AKS_NAME=myWanderblogAks
```

2. Create resource group, ACR, and AKS (example)

```powershell
az group create -n $RESOURCE_GROUP -l $LOCATION
az acr create -n $ACR_NAME -g $RESOURCE_GROUP --sku Standard
az aks create -n $AKS_NAME -g $RESOURCE_GROUP --node-count 2 --generate-ssh-keys --attach-acr $ACR_NAME
az aks get-credentials -n $AKS_NAME -g $RESOURCE_GROUP
```

3. Build and push images (or use `az acr build`)

```powershell
# Build locally then push
docker build -t $ACR_NAME.azurecr.io/wanderblog-frontend:latest ./frontend
docker build -t $ACR_NAME.azurecr.io/wanderblog-backend:latest ./server
docker push $ACR_NAME.azurecr.io/wanderblog-frontend:latest
docker push $ACR_NAME.azurecr.io/wanderblog-backend:latest

# OR use Azure Build
az acr build -t wanderblog-frontend:latest -r $ACR_NAME ./frontend
az acr build -t wanderblog-backend:latest -r $ACR_NAME ./server
```

4. Install ingress-nginx and cert-manager

```powershell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

5. Create TLS certificate for `wanderblog.online` (use cert-manager Issuer/ClusterIssuer) and ensure a `tls-wanderblog` secret exists, or remove TLS from `kubernetes-manifests/ingress.yml` for testing.

6. Update image references in manifests if needed to point to `$ACR_NAME.azurecr.io/...` and apply manifests:

```powershell
kubectl apply -f kubernetes-manifests/storage-class.yml
kubectl apply -f kubernetes-manifests/mongo-sts.yml
kubectl apply -f kubernetes-manifests/mongo-service.yml
kubectl apply -f kubernetes-manifests/backend.yml
kubectl apply -f kubernetes-manifests/backend-service.yml
kubectl apply -f kubernetes-manifests/frontend.yml
kubectl apply -f kubernetes-manifests/frontend-service.yml
kubectl apply -f kubernetes-manifests/ingress.yml
```

Notes:
- The `storage-class.yml` was changed to use Azure Disk CSI (`disk.csi.azure.com`).
- The `ingress.yml` was changed to be compatible with `ingress-nginx` (remove AWS ALB annotations).
- If you prefer Application Gateway, install AGIC and adjust annotations accordingly.
