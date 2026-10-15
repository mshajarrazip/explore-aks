# Kubernetes manifest primer

Kubernetes config files (manifests) are YAML descriptions of objects you want to exist in the cluster. You declare the desired state, `kubectl apply` sends it to the API server, and Kubernetes keeps working to make the cluster match it.

Almost every manifest has the same four top-level fields:

| Field | What it is | In [`module4/deployment.yaml`](../module4/deployment.yaml) |
|---|---|---|
| `apiVersion` | API group and version that defines the object type | `apps/v1` |
| `kind` | Type of object | `Deployment` |
| `metadata` | Identity of the object: `name`, optional `namespace`, `labels`, `annotations` | `name: my-app` |
| `spec` | Desired state; its fields depend on the `kind` | replicas, selector, pod template |

After you apply a manifest, Kubernetes adds a `status` field with the actual state. You never write `status` yourself; see it with `kubectl get deployment my-app -o yaml`.

## Labels and selectors

Labels are key/value tags on objects (`app: my-app`). Selectors pick objects by label, and this is how objects find each other:

- A Deployment's `spec.selector` must match its `spec.template.metadata.labels`, so it knows which pods it owns.
- A Service selects pods the same way to route traffic to them.
- `kubectl get pods -l app=my-app` filters by label.

## Nested specs

A Deployment doesn't run containers directly. Its `spec.template` is a pod template, which has its own `metadata` and `spec`. The chain is:

Deployment → ReplicaSet (created for you) → Pods → containers

so `module4/deployment.yaml` has two `metadata`/`spec` pairs: one for the Deployment and one for the pods it creates.

## Useful commands

```bash
kubectl apply -f deployment.yaml      # create or update the object
kubectl diff -f deployment.yaml       # preview what apply would change
kubectl explain deployment.spec       # built-in docs for any field
kubectl explain deployment.spec.template.spec.containers
kubectl get deployment my-app -o yaml # full object as stored, including status
kubectl delete -f deployment.yaml     # delete what the file describes
```

Several objects can share one file, separated by a line containing `---`.
