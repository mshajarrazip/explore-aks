# kubectl cheatsheet

Everyday `kubectl` commands for AKS, minikube and CKAD practice. See [minikube-cheatsheet.md](./minikube-cheatsheet.md) for the local cluster and [manifest-primer.md](./manifest-primer.md) for how manifests are structured.

## Setup and shortcuts

```bash
alias k=kubectl
source <(kubectl completion bash)       # tab completion
complete -o default -F __start_kubectl k   # completion for the alias too
export do="--dry-run=client -o yaml"    # k create deploy x --image=nginx $do > x.yaml
export now="--force --grace-period=0"   # k delete pod x $now

kubectl version                         # client and server versions
kubectl cluster-info                    # API server and core service endpoints
```

## Contexts and namespaces

```bash
kubectl config get-contexts             # list clusters/users in kubeconfig
kubectl config current-context
kubectl config use-context minikube
kubectl config use-context aks-hajar-dev

kubectl config set-context --current --namespace=dev   # default namespace for this context
kubectl config view --minify            # only the active context

az aks get-credentials -g $RESOURCE_GROUP -n aks-hajar-dev   # add an AKS cluster to kubeconfig

kubectl get ns
kubectl create ns dev
kubectl get pods -n dev                 # one namespace
kubectl get pods -A                     # all namespaces
```

## Viewing resources

```bash
kubectl get pods                        # also: deploy, svc, rs, cm, secret, ing, pv, pvc, sa, nodes
kubectl get pods -o wide                # node and pod IP
kubectl get pods -o yaml                # full object (also -o json)
kubectl get pods -w                     # watch for changes
kubectl get all                         # pods, services, deployments, replicasets
kubectl get pods --show-labels
kubectl get pods -l app=my-app          # filter by label (also -l 'env in (dev,test)')
kubectl get pods --field-selector status.phase=Running
kubectl get pods --sort-by=.metadata.creationTimestamp

kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName

kubectl describe pod my-pod             # details + events (first stop when debugging)
kubectl get events --sort-by=.lastTimestamp

kubectl api-resources                   # every kind, short name and API group
kubectl explain pod.spec.containers     # field docs (add --recursive for the whole tree)
```

## Creating resources imperatively

Fast in the exam; add `$do > file.yaml` to generate a manifest instead of creating the object.

```bash
kubectl run nginx --image=nginx                         # pod
kubectl run nginx --image=nginx --port=80 --labels=app=web
kubectl run busybox --image=busybox --restart=Never -- sleep 3600
kubectl run tmp --image=busybox --rm -it --restart=Never -- sh   # throwaway shell

kubectl create deployment my-app --image=nginx --replicas=3
kubectl create job hello --image=busybox -- echo hi
kubectl create cronjob hello --image=busybox --schedule="*/5 * * * *" -- echo hi
kubectl create job manual-run --from=cronjob/hello     # trigger a cronjob now

kubectl create configmap app-cfg --from-literal=MODE=dev --from-file=config.conf
kubectl create secret generic db --from-literal=password=s3cret
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key
kubectl create serviceaccount mysa

kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding pod-reader-b --role=pod-reader --serviceaccount=dev:mysa
kubectl create clusterrole node-reader --verb=get,list --resource=nodes
kubectl create clusterrolebinding node-reader-b --clusterrole=node-reader --user=anna

kubectl create ingress web --rule="example.com/=my-app:80"
kubectl create quota dev-quota --hard=pods=10,cpu=2,memory=4Gi -n dev
```

## Applying and editing manifests

```bash
kubectl apply -f deployment.yaml        # create or update (declarative)
kubectl apply -f ./manifests/           # whole directory (-R for subdirs)
kubectl apply -k ./kustomize-demo       # kustomize overlay
kubectl diff -f deployment.yaml         # preview changes before apply
kubectl create -f pod.yaml              # create only; fails if it exists
kubectl replace --force -f pod.yaml     # delete and recreate (for immutable pod fields)

kubectl edit deploy my-app              # edit live object in $EDITOR
kubectl get deploy my-app -o yaml > my-app.yaml   # export to tweak and re-apply

kubectl label pod my-pod env=dev        # add/change label (--overwrite to change)
kubectl label pod my-pod env-           # remove label
kubectl annotate pod my-pod note="hello"
kubectl patch deploy my-app -p '{"spec":{"replicas":2}}'
kubectl set image deploy/my-app nginx=nginx:1.27
kubectl set env deploy/my-app MODE=prod
kubectl set resources deploy/my-app --requests=cpu=100m,memory=128Mi --limits=memory=256Mi
kubectl set serviceaccount deploy/my-app mysa
```

## Deployments, scaling and rollouts

```bash
kubectl scale deploy my-app --replicas=5
kubectl autoscale deploy my-app --min=2 --max=5 --cpu-percent=70   # HPA (needs metrics-server)

kubectl rollout status deploy/my-app
kubectl rollout history deploy/my-app
kubectl rollout history deploy/my-app --revision=2
kubectl rollout undo deploy/my-app                  # back to previous revision
kubectl rollout undo deploy/my-app --to-revision=2
kubectl rollout restart deploy/my-app               # recreate pods (e.g. after a configmap change)
kubectl rollout pause deploy/my-app / kubectl rollout resume deploy/my-app
```

## Services and networking

```bash
kubectl expose deploy my-app --port=80 --target-port=8000                 # ClusterIP
kubectl expose deploy my-app --type=NodePort --port=80 --target-port=8000
kubectl expose deploy my-app --type=LoadBalancer --port=80 --target-port=8000   # public IP on AKS
kubectl create service clusterip my-svc --tcp=80:8000

kubectl get svc,endpoints               # endpoints empty = selector doesn't match pods
kubectl port-forward svc/my-app 8080:80 # localhost:8080 -> service
kubectl port-forward pod/my-pod 8080:8000

kubectl run tmp --image=busybox --rm -it --restart=Never -- wget -qO- http://my-app   # test from inside
kubectl run tmp --image=busybox --rm -it --restart=Never -- nslookup my-app.dev.svc.cluster.local
```

## Debugging pods

```bash
kubectl logs my-pod                     # stdout of the (only) container
kubectl logs my-pod -c sidecar          # specific container
kubectl logs my-pod -f                  # follow
kubectl logs my-pod --previous          # last crashed container (CrashLoopBackOff)
kubectl logs deploy/my-app --tail=50
kubectl logs -l app=my-app --all-containers

kubectl exec -it my-pod -- sh           # shell (bash if the image has it)
kubectl exec my-pod -c sidecar -- cat /etc/config/app.conf
kubectl cp my-pod:/tmp/out.txt ./out.txt
kubectl debug -it my-pod --image=busybox --target=app   # ephemeral debug container

kubectl top pods                        # CPU/memory (needs metrics-server)
kubectl top nodes
kubectl auth can-i list pods --as=system:serviceaccount:dev:mysa   # check RBAC
```

## Nodes and scheduling

```bash
kubectl get nodes -o wide
kubectl describe node <node>            # capacity, allocations, taints, conditions
kubectl label node <node> disktype=ssd  # for nodeSelector / node affinity
kubectl taint node <node> key=value:NoSchedule
kubectl taint node <node> key=value:NoSchedule-   # remove taint

kubectl cordon <node>                   # stop scheduling new pods
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data   # evict pods for maintenance
kubectl uncordon <node>
```

## Deleting

```bash
kubectl delete -f deployment.yaml
kubectl delete pod my-pod
kubectl delete pod my-pod --force --grace-period=0   # immediate ($now)
kubectl delete deploy,svc -l app=my-app
kubectl delete ns dev                   # deletes everything in it
```

## Pod status troubleshooting

| Status / symptom | Usually means | Check |
|---|---|---|
| `Pending` | no node fits (resources, taints, affinity, unbound PVC) | `kubectl describe pod` events |
| `ImagePullBackOff` / `ErrImagePull` | wrong image name/tag or registry auth | `describe pod`; on AKS `az aks check-acr` |
| `CrashLoopBackOff` | app exits on start | `kubectl logs --previous` |
| `CreateContainerConfigError` | missing ConfigMap/Secret key | `describe pod`, `get cm,secret` |
| `OOMKilled` | memory limit too low | `describe pod` last state; raise limits |
| Running but not `Ready` | readiness probe failing | `describe pod` probe events |
| Service unreachable | selector/port mismatch | `get endpoints`, compare labels and `targetPort` |
| Command hits wrong cluster | wrong context | `kubectl config current-context` |
