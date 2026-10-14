# Exploring Azure Kubernetes Services

Previously, I did some tutorials on K8s fundamentals: [Notes here](https://github.com/mshajarrazip/learning-k8s)

Using the Couresera tutorial: [Deploy containers by using Azure Kubernetes Service](https://www.coursera.org/learn/deploy-containers-with-azure-kubernetes-service/home/welcome)

## Log 

2026-10-14: Will set up terraform next to create temporary resources for the purpose of this demo

## Notes

1. Create managed node resource group

```bash
RESOURCE_GROUP=rg-name
LOCATION=$(az group show --name $RESOURCE_GROUP --query location -o tsv)

az aks create \
  --resource-group $RESOURCE_GROUP \
  --name aks-hajar-dev \
  --node-resource-group rg-aks-hajar-nodes \
  --location $LOCATION \
  --tier free \
  --node-count 1 \
  --node-vm-size Standard_D2s_v3 \
  --generate-ssh-keys \
  --tags project=aks-experiment environment=dev
```

1. Check cluster:

```bash
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name aks-hajar-dev \
  --query "{name:name, state:provisioningState, tier:sku.tier, nodeRG:nodeResourceGroup}" \
  -o table
```

1. Set up `kubectl`

```bash
# merges with current context
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name aks-hajar-dev

kubectl config get-contexts
kubectl config use-context aks-hajar-dev
kubectl get nodes

# additional commands
kubectl cluster-info
kubectl get namespaces
kubectl get pods -A
```