# Module 4: Deploy a container to AKS

Files:

- `Dockerfile`: builds a Node.js app image (expects `package.json` and app source in this folder)
- `compose.yaml`: runs the image locally, equivalent to `deployment.yaml`
- `deployment.yaml`: Kubernetes Deployment `my-app` with 1 replica

## Run locally

1. Build and start the container

```bash
docker compose up --build -d
docker compose ps
docker compose logs -f my-app
```

1. Stop and clean up

```bash
docker compose down
```

## Deploy to AKS

AKS can't pull `my-docker-image:latest` from your local Docker, so the image needs to be pushed to a registry first (e.g. Azure Container Registry).

1. Create an ACR and attach it to the cluster

```bash
RESOURCE_GROUP=rg-name
ACR_NAME=acrhajardev  # must be globally unique, alphanumeric only

az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Basic

az aks update \
  --resource-group $RESOURCE_GROUP \
  --name aks-hajar-dev \
  --attach-acr $ACR_NAME
```

1. Build and push the image

```bash
az acr login --name $ACR_NAME

docker build -t $ACR_NAME.azurecr.io/my-docker-image:latest .
docker push $ACR_NAME.azurecr.io/my-docker-image:latest

# or build directly in Azure without local Docker
# az acr build --registry $ACR_NAME --image my-docker-image:latest .
```

1. Point `deployment.yaml` at the pushed image

```yaml
image: <ACR_NAME>.azurecr.io/my-docker-image:latest
```

1. Apply the deployment

```bash
kubectl config use-context aks-hajar-dev
kubectl apply -f deployment.yaml

kubectl get deployments
kubectl get pods -l app=my-app
kubectl logs -l app=my-app
```

1. Access the app (no Service is defined yet, so port-forward to test)

```bash
kubectl port-forward deployment/my-app 3000:3000
# then open http://localhost:3000
```

1. Clean up

```bash
kubectl delete -f deployment.yaml
```
