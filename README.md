# Exploring Azure Kubernetes Services

## Notes

1. Create managed node resource group

```bash
RESOURCE_GROUP=rg-name
LOCATION=$(az group show --name $RESOURCE_GROUP --query location -o tsv)

az aks create \
  --resource-group $RESOURCE_GROUP \
  --name aks-hajar-dev \
  --node-resource-group rg-aks-hajar-nodes \
  --location malaysiawest \
  --tier free \
  --node-count 1 \
  --node-vm-size Standard_D2s_v3 \
  --generate-ssh-keys \
  --tags project=aks-experiment environment=dev
```

1. Check cluster:

```bash
az aks show \
  --resource-group rg-cerdas-dev \
  --name aks-hajar-dev \
  --query "{name:name, state:provisioningState, tier:sku.tier, nodeRG:nodeResourceGroup}" \
  -o table
```