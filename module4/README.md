# Module 4: Deploy a container to AKS

Files:

- `main.py`: FastAPI app with a single `GET /` endpoint
- `requirements.txt`: Python dependencies
- `Dockerfile`: builds the FastAPI app image (served by uvicorn on port 8000)
- `compose.yaml`: runs the image locally, equivalent to `deployment.yaml`
- `deployment.yaml`: Kubernetes Deployment `my-app` with 1 replica (commented line by line)

See [docs/manifest-primer.md](../docs/manifest-primer.md) for the basics of Kubernetes config files.

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

AKS can't pull `my-docker-image:latest` from your local Docker, so the image needs to be pushed to a registry first. This uses GitHub Container Registry (`ghcr.io`).

1. Log in to GitHub with the `gh` CLI, requesting the `write:packages` scope

```bash
gh auth login --scopes write:packages
# or, if you're already logged in:
# gh auth refresh --scopes write:packages
```

1. Build and push the image

```bash
GH_USER=username # must be lowercase in the image name

gh auth token | docker login ghcr.io -u $GH_USER --password-stdin

# builds and pushes the image named in compose.yaml
docker compose build --push
```

1. Make the package public so AKS can pull it without credentials: on GitHub go to your profile → Packages → `my-docker-image` → Package settings → Change visibility → Public

1. `deployment.yaml` already points at the pushed image

```yaml
image: ghcr.io/mshajarrazip/my-docker-image:latest
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
kubectl port-forward deployment/my-app 8000:8000
# then open http://localhost:8000
```

1. Clean up

```bash
kubectl delete -f deployment.yaml
```
