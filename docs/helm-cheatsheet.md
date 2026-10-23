# Helm cheatsheet

Helm is the package manager for Kubernetes: a **chart** is a package of templated manifests, and installing it creates a **release** in the cluster. Helm talks to whatever cluster your kubectl context points at (see [kubectl-cheatsheet.md](./kubectl-cheatsheet.md)). [`ckad/my-ghost-app`](../ckad/my-ghost-app) is a chart made with `helm create` to experiment with.

Commands below are for Helm 4 (installed locally: v4.3.1). Helm 3 syntax still works with deprecation warnings; see [Helm 3 vs Helm 4](#helm-3-vs-helm-4).

## Concepts

| Term | What it is |
|---|---|
| Chart | Directory or `.tgz` with `Chart.yaml`, `values.yaml` and `templates/` |
| Repository | Server hosting charts (`helm repo add`), or an OCI registry (`oci://...`) |
| Release | One installed instance of a chart, with a name, scoped to a namespace |
| Revision | Version of a release; each install/upgrade/rollback adds one |
| Values | Settings that fill the templates: chart defaults < `-f` files < `--set` |

Release history is stored as Secrets in the release's namespace (`sh.helm.release.v1.<release>.v<revision>`), which is why `helm list` only shows the current namespace unless you pass `-A`.

## Setup

```bash
brew install helm                       # macOS
sudo snap install helm --classic        # Ubuntu (what the CKAD course uses)

helm version
source <(helm completion zsh)           # tab completion (bash/fish/powershell too)
helm env                                # cache, config and plugin paths
```

## Repositories and searching

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo list
helm repo update                        # refresh indexes (do this before searching/installing)
helm repo remove headlamp

helm search repo nginx                  # search added repos
helm search repo bitnami/nginx --versions   # every chart version (CHART VERSION vs APP VERSION)
helm search hub wordpress               # search Artifact Hub (artifacthub.io)
```

OCI registries need no `repo add`; reference the chart directly:

```bash
helm show values oci://registry-1.docker.io/bitnamicharts/redis
helm install my-redis oci://registry-1.docker.io/bitnamicharts/redis --version 20.0.0
```

## Inspecting a chart before installing

```bash
helm show values bitnami/nginx          # default values: the knobs you can turn
helm show values bitnami/nginx | grep -A3 replicaCount
helm show values bitnami/nginx > values.yaml   # copy, edit, then install with -f
helm show chart bitnami/mysql           # Chart.yaml metadata (version, appVersion, deps)
helm show readme bitnami/mysql | less
helm show all bitnami/mysql | less

helm pull bitnami/nginx --untar         # download and unpack to read the templates
helm pull bitnami/nginx --version 18.1.0   # just the .tgz
```

## Installing

```bash
helm install webserver bitnami/nginx                  # release name + chart
helm install bitnami/mysql --generate-name            # auto name, e.g. mysql-1770116180
helm install my-mariadb bitnami/mariadb --version 25.0.10   # pin chart version
helm install webserver bitnami/nginx -n web --create-namespace

helm install webserver bitnami/nginx -f values.yaml   # override with a values file
helm install redis bitnami/redis --set replicaCount=3 # override one value
helm install webserver bitnami/nginx -f base.yaml -f prod.yaml --set image.tag=1.27   # later wins

helm install webserver bitnami/nginx --dry-run=client # render and print, change nothing
helm install webserver bitnami/nginx --wait --timeout 5m   # block until resources are ready
helm install webserver bitnami/nginx --rollback-on-failure # uninstall again if it fails (implies --wait)

helm upgrade --install webserver bitnami/nginx        # install or upgrade: safe to re-run (scripts, CI)
```

## Setting values

```bash
--set replicaCount=3                    # top-level key
--set service.type=NodePort             # nested: service: {type: NodePort}
--set commonLabels.env=dev
--set ingress.hosts[0].host=example.com # list element
--set args={--verbose,--port=8080}      # whole list
--set name=a\,b                         # escape commas
--set-string image.tag=1.20             # force string (otherwise 1.20 becomes a number)
--set-file config=./nginx.conf          # value from a file's contents
--set-json 'resources={"limits":{"memory":"256Mi"}}'
```

Precedence, lowest to highest: chart `values.yaml` < parent chart values (for subcharts) < `-f` files in order < `--set*` flags.

## Working with releases

```bash
helm list                               # releases in current namespace (alias: ls)
helm list -A                            # all namespaces
helm list -n web --failed               # also --pending, --deployed, --uninstalled
helm list -q | grep mysql               # names only (handy in scripts)

helm status webserver                   # state, revision, NOTES (how to reach the app)
helm get values webserver               # only the values you overrode
helm get values webserver --all         # every computed value
helm get values webserver --revision 1
helm get manifest webserver             # rendered YAML that was applied
helm get notes webserver
helm get all webserver
helm history webserver                  # revisions with status and description

kubectl get all -l app.kubernetes.io/instance=webserver   # objects created by the release
helm get manifest webserver | kubectl get -f -            # same, without relying on labels
```

## Upgrading and rolling back

```bash
helm repo update
helm upgrade webserver bitnami/nginx --set replicaCount=3
helm upgrade webserver bitnami/nginx -f values.yaml --version 18.2.0
helm upgrade webserver bitnami/nginx --reuse-values --set image.tag=1.27   # keep previous overrides
helm upgrade webserver bitnami/nginx --reset-values                       # back to chart defaults
helm upgrade webserver bitnami/nginx --rollback-on-failure --timeout 3m   # auto rollback if not ready

helm history webserver
helm rollback webserver                 # to previous revision
helm rollback webserver 1               # to a specific revision (creates a new revision)
```

Gotcha: an upgrade with no `-f`/`--set` keeps the previous values, but as soon as you pass any override, earlier overrides are dropped. Re-pass them, use the same values file, or add `--reuse-values`.

## Uninstalling

```bash
helm uninstall webserver                # also: helm delete / helm del / helm un
helm uninstall webserver -n web
helm uninstall webserver --keep-history # keep history, release shows in `helm list --uninstalled`
helm uninstall webserver --wait         # block until resources are gone
```

PersistentVolumeClaims created by StatefulSets (e.g. databases) are not deleted; clean them up with `kubectl delete pvc -l app.kubernetes.io/instance=<release>`.

## Rendering and debugging

```bash
helm template webserver bitnami/nginx -f values.yaml          # render locally, no cluster
helm template demo ./ckad/my-ghost-app -s templates/deployment.yaml   # one template only
helm template demo ./ckad/my-ghost-app --set replicaCount=3 | grep replicas
helm template webserver bitnami/nginx > rendered.yaml        # inspect or `kubectl apply -f` it

helm lint ./ckad/my-ghost-app           # chart structure and template errors
helm install demo ./ckad/my-ghost-app --dry-run=server --debug   # validate against the API server
helm test demo                          # run the chart's templates/tests/ pods
```

## Writing a chart

```bash
helm create my-ghost-app                # scaffold (nginx-based Deployment, Service, Ingress, HPA...)
helm lint my-ghost-app
helm install ghost ./my-ghost-app       # install from the directory
helm upgrade ghost ./my-ghost-app --set replicaCount=2

helm package my-ghost-app               # -> my-ghost-app-0.1.0.tgz (version from Chart.yaml)
helm install ghost ./my-ghost-app-0.1.0.tgz
helm dependency update my-ghost-app     # download charts listed under dependencies into charts/
```

```text
my-ghost-app/
├── Chart.yaml          # name, version (chart), appVersion (app), dependencies
├── values.yaml         # defaults for .Values
├── charts/             # subcharts (from helm dependency update)
└── templates/
    ├── _helpers.tpl    # named templates (fullname, labels); _ files render nothing
    ├── deployment.yaml
    ├── service.yaml
    ├── NOTES.txt       # printed after install/upgrade
    └── tests/          # pods run by `helm test`
```

Bump `version` in `Chart.yaml` whenever the chart changes; `appVersion` is the app's version and is the usual default image tag.

Template syntax (Go templates + Sprig functions):

```yaml
metadata:
  name: {{ include "my-ghost-app.fullname" . }}       # named template from _helpers.tpl
  labels:
    {{- include "my-ghost-app.labels" . | nindent 4 }}
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          env:
            {{- range .Values.env }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
{{- if .Values.ingress.enabled }}
# ...only rendered when ingress.enabled=true
{{- end }}
```

| Built-in | Value |
|---|---|
| `.Values.x` | merged values |
| `.Release.Name` / `.Release.Namespace` / `.Release.Revision` | release info |
| `.Chart.Name` / `.Chart.Version` / `.Chart.AppVersion` | from `Chart.yaml` |
| `.Capabilities.KubeVersion` | cluster version (for API compatibility checks) |
| `.Files.Get "conf/app.conf"` | contents of a non-template file in the chart |

`{{-` / `-}}` trim whitespace before/after; `nindent N` adds a newline and indents, which keeps YAML valid when inserting blocks.

## Charts on Azure Container Registry

```bash
ACR_NAME=myacr
PASSWORD=$(az acr login --name $ACR_NAME --expose-token --output tsv --query accessToken)
helm registry login $ACR_NAME.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password $PASSWORD

helm package my-ghost-app
helm push my-ghost-app-0.1.0.tgz oci://$ACR_NAME.azurecr.io/helm
helm install ghost oci://$ACR_NAME.azurecr.io/helm/my-ghost-app --version 0.1.0
```

## CKAD quick loop

The exam tasks are usually "add a repo, install a chart (with a value), check it, change it":

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update
helm search repo bitnami/mysql
helm show values bitnami/mysql | less   # find the key you need to set
helm install bitnami/mysql --generate-name
helm list
helm status mysql-<id>
kubectl get pods

helm install webserver bitnami/nginx --set replicaCount=2
helm upgrade webserver bitnami/nginx --set replicaCount=3
helm history webserver
helm rollback webserver 1
helm uninstall webserver
```

Fall back to `helm <command> --help`; the examples at the top of each help page are often exactly what the task asks.

## Helm 3 vs Helm 4

The CKAD environment may run Helm 3. Old flags still work on Helm 4 with a deprecation warning, so Helm 3 syntax is safe on both.

| Helm 3 | Helm 4 |
|---|---|
| `--atomic` | `--rollback-on-failure` |
| `--force` | `--force-replace` |
| `--wait` (legacy readiness checks) | `--wait` uses the `watcher` strategy; `--wait=legacy` for the old behavior |
| client-side apply | server-side apply for new installs (`--server-side=false` to opt out); conflicts need `--force-conflicts` |

## Troubleshooting

| Symptom | Usually means | Try |
|---|---|---|
| `cannot re-use a name that is still in use` | release name already exists in the namespace | `helm list -A`; pick another name or `helm upgrade --install` |
| `repo ... not found` / `chart ... not found` | repo not added or index stale | `helm repo add ...`, `helm repo update`, `helm search repo` |
| Release missing from `helm list` | it's in another namespace | `helm list -A`, then add `-n <ns>` |
| `another operation (install/upgrade/rollback) is in progress` | earlier command was interrupted, release stuck `pending-*` | `helm history <rel>`, then `helm rollback <rel> <last deployed rev>` |
| `--set` seems ignored | wrong key path | `helm show values <chart>`, then check with `helm get values <rel> --all` |
| Upgrade lost earlier settings | overrides aren't carried over when you pass new ones | re-pass `-f values.yaml` or use `--reuse-values` |
| Template / YAML parse error | bad indentation or missing value | `helm template --debug`, `helm lint` |
| `invalid ownership metadata` | resource already exists outside this release | delete it, or install with `--take-ownership` |
| Bitnami pods in `ImagePullBackOff` | Bitnami moved most free images to `docker.io/bitnamilegacy` in 2025 | check the image in `kubectl describe pod`; override `image.repository` (Bitnami charts also want `--set global.security.allowInsecureImages=true`) |
| Pods not ready after install | app problem, not Helm | `kubectl describe pod`, `kubectl logs` (see kubectl cheatsheet) |
