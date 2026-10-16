# minikube cheatsheet

minikube runs a local Kubernetes cluster for practice before deploying to AKS. Your normal `kubectl` commands work against it once its context is active.

## Cluster lifecycle

```bash
minikube start                         # create/start the default cluster ("minikube" profile)
minikube start --driver=docker         # pick a driver (docker is the usual choice on WSL2)
minikube start --cpus=2 --memory=4g    # set resources (only applies when the cluster is created)
minikube start --kubernetes-version=v1.31.0   # pin a Kubernetes version
minikube start --nodes=2               # multi-node cluster

minikube status                        # host / kubelet / apiserver state
minikube stop                          # stop, keep state
minikube pause / minikube unpause      # freeze workloads without stopping the VM/container
minikube delete                        # delete the cluster
minikube delete --all --purge          # delete every profile and ~/.minikube
```

## Profiles (multiple clusters)

```bash
minikube start -p ckad                 # create a second cluster named "ckad"
minikube profile list                  # list clusters
minikube profile ckad                  # make "ckad" the default for minikube commands
minikube stop -p ckad                  # -p works with most commands
```

## kubectl and contexts

`minikube start` sets your kubectl context to `minikube`. After working on AKS, switch back:

```bash
kubectl config get-contexts
kubectl config use-context minikube
kubectl config use-context aks-hajar-dev   # back to AKS

minikube kubectl -- get pods -A        # bundled kubectl matching the cluster version
minikube update-context                # fix kubeconfig if the IP/port changed
```

## Accessing services

```bash
minikube service my-app                # open a NodePort/LoadBalancer service in the browser
minikube service my-app --url          # just print the URL
minikube service list                  # all services and their URLs

minikube tunnel                        # give LoadBalancer services an external IP (keep running)

kubectl port-forward svc/my-app 8080:80   # works without minikube helpers too

minikube ip                            # node IP (with the docker driver on WSL2, prefer service/tunnel)
```

## Images

Pods pull images from inside the cluster, so local `docker build` images aren't visible by default.

```bash
minikube image load my-app:dev         # copy a local image into the cluster
minikube image build -t my-app:dev .   # build directly inside the cluster
minikube image ls                      # images available in the cluster
minikube image rm my-app:dev

eval $(minikube docker-env)            # point your shell's docker CLI at minikube's daemon
eval $(minikube docker-env -u)         # undo
```

Set `imagePullPolicy: IfNotPresent` (or `Never`) in the manifest so Kubernetes uses the loaded image instead of pulling `:latest` from a registry.

## Addons

```bash
minikube addons list
minikube addons enable ingress         # NGINX ingress controller
minikube addons enable metrics-server  # enables `kubectl top`
minikube addons enable dashboard
minikube addons disable ingress
minikube dashboard                     # open the Kubernetes dashboard (--url to print it)
```

## Node access, files and logs

```bash
minikube ssh                           # shell on the node
minikube ssh -- docker ps              # run one command on the node
minikube cp ./local.txt /home/docker/local.txt   # copy a file to the node
minikube mount ./data:/data            # mount a host dir into the node (keep running)

minikube logs                          # cluster logs
minikube logs -f                       # follow
minikube logs --file=logs.txt          # save for debugging
```

## Config defaults

```bash
minikube config set driver docker
minikube config set memory 4096
minikube config set cpus 2
minikube config view
minikube config unset memory
```

## Quick loop: deploy module4 locally

```bash
minikube start
kubectl apply -f module4/deployment.yaml
kubectl get pods -w
kubectl expose deployment my-app --type=NodePort --port=80 --target-port=8000
minikube service my-app
kubectl delete -f module4/deployment.yaml && kubectl delete svc my-app
```

## Troubleshooting

| Symptom | Try |
|---|---|
| `minikube start` hangs or fails | `minikube delete` then start again; check `minikube logs` |
| Pod stuck in `ErrImagePull` for a local image | `minikube image load <img>` and set `imagePullPolicy: IfNotPresent` |
| Service URL doesn't open on WSL2 | use `minikube service <svc> --url` and keep the terminal open, or `kubectl port-forward` |
| LoadBalancer `EXTERNAL-IP` stays `<pending>` | run `minikube tunnel` in another terminal |
| kubectl talks to the wrong cluster | `kubectl config current-context`, then `use-context minikube` |
| `kubectl top` errors | `minikube addons enable metrics-server` and wait a minute |
