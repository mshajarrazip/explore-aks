# Exploring Azure Kubernetes Services

Previously, I did some tutorials on K8s fundamentals: [Notes here](https://github.com/mshajarrazip/learning-k8s)

[module4](./module4) contain some hands-on while I followed the Couresera course: [Deploy containers by using Azure Kubernetes Service](https://www.coursera.org/learn/deploy-containers-with-azure-kubernetes-service/home/welcome). I've Completed this course: [ See my certificate ](https://coursera.org/share/07dbfe89159dd36138cf5f815832aea2)

Now, doing Coursera's [Certified Kubernetes Application Developer (CKAD)](https://www.coursera.org/specializations/pearson-certified-kubernetes-application-developer-ckad) course for a bit more mastery.

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
1. Delete the cluster when done

```bash
# also deletes the managed node resource group (rg-aks-hajar-nodes) and everything in it;
# $RESOURCE_GROUP itself is kept
az aks delete \
  --resource-group $RESOURCE_GROUP \
  --name aks-hajar-dev \
  --yes

# remove the stale entries from your kubeconfig
kubectl config delete-context aks-hajar-dev
kubectl config delete-cluster aks-hajar-dev
kubectl config delete-user clusterUser_${RESOURCE_GROUP}_aks-hajar-dev
```

To pause instead of delete (stops node VM charges, keeps the cluster config):

```bash
az aks stop --resource-group $RESOURCE_GROUP --name aks-hajar-dev
az aks start --resource-group $RESOURCE_GROUP --name aks-hajar-dev
```
