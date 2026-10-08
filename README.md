# DSO202 Unit III 3.2 — Helm: Practical Report

**Module:** DSO202 · **Unit:** III 3.2 — Helm
**Tools:** kind (3-node cluster), kubectl, Helm 4, Homebrew (macOS)
**Reference guide:** [Unit III 3.2 — Helm](https://hackmd.io/RwJPCvlnR727QEgZI_zQYg)

---

## Table of Contents

1. [Objectives](#1-objectives)
2. [Lab Environment](#2-lab-environment)
3. [Stage 0 — Installing Helm and Preparing the Cluster](#3-stage-0--installing-helm-and-preparing-the-cluster)
4. [Stage 1 — Consuming a Published Chart](#4-stage-1--consuming-a-published-chart)
5. [Stage 2 — Chart Structure and Chart.yaml](#5-stage-2--chart-structure-and-chartyaml)
6. [Stage 3 — Writing the webapp Templates](#6-stage-3--writing-the-webapp-templates)
7. [Stage 4 — Values: Sources, Precedence and Merging](#7-stage-4--values-sources-precedence-and-merging)
8. [Stage 5 — Validating Charts Before Installation](#8-stage-5--validating-charts-before-installation)
9. [Summary of Results](#9-summary-of-results)
10. [Conclusion](#10-conclusion)

---

## 1. Objectives

By the end of this practical I aimed to:

- install Helm 4 and connect it to a local kind cluster;
- consume a published chart (podinfo): search, inspect, install, upgrade, roll back and uninstall a release;
- build my own chart (`webapp`) from scratch, including `Chart.yaml`, `values.yaml`, templates and helpers;
- understand how values from different sources are merged and which one wins;
- validate a chart before installation using `helm lint` and `helm template`.

---

## 2. Lab Environment

| Item | Value |
| --- | --- |
| Cluster | kind, 3 nodes (1 control plane, 2 workers) from Practical 1 |
| Cluster config | `cluster/kind-cluster.yaml` (Listing 1, Practical 1) |
| Helm | v4.x, installed with Homebrew |
| Demo namespace (Stage 1) | `dso202-helm` |
| Lab directory | `dso202-helm-lab/` (under version control) |

Final lab directory layout:

```text
dso202-helm-lab/
├── environments/        # per-environment values files
│   ├── dev.yaml
│   └── prod.yaml
└── webapp/              # chart written in Stages 2–5
    ├── Chart.yaml
    ├── values.yaml
    ├── .helmignore
    ├── charts/
    └── templates/
        ├── _helpers.tpl
        ├── configmap.yaml
        ├── deployment.yaml
        ├── service.yaml
        ├── NOTES.txt
        └── tests/
            └── test-connection.yaml
```

---

## 3. Stage 0 — Installing Helm and Preparing the Cluster

**Purpose:** confirm that the cluster and the Helm binary each work on their own, so any later failure can be traced to a single cause.

### Step 0.1 — Start the kind cluster

```bash
kind create cluster --config cluster/kind-cluster.yaml
kubectl config current-context
kubectl get nodes
```

![Figure 1 — kind cluster created; three nodes in Ready state](evidence/1.png)
*Figure 1 — The kind cluster is running and all three nodes report `Ready`.*

### Step 0.2 — Install Helm 4

Helm was installed on macOS with Homebrew (official instructions: <https://helm.sh/docs/intro/install/>).

```bash
brew install helm
```

![Figure 2 — Helm installed with Homebrew](evidence/2.png)
*Figure 2 — Helm installed successfully.*

### Step 0.3 — Inspect Helm's local configuration

Helm keeps three important local paths:

| Variable | Stores |
| --- | --- |
| `HELM_REPOSITORY_CONFIG` | the list of added chart repositories |
| `HELM_REPOSITORY_CACHE` | downloaded repository indexes |
| `HELM_REGISTRY_CONFIG` | OCI registry credentials |

```bash
helm env | grep -E 'HELM_(NAMESPACE|MAX_HISTORY|REPOSITORY_CONFIG|REPOSITORY_CACHE|REGISTRY_CONFIG)'
```

![Figure 3 — helm env output](evidence/3.png)
*Figure 3 — Helm's configuration paths. `HELM_MAX_HISTORY="10"` means at most ten revisions are kept per release.*

### Step 0.4 — Confirm Helm can reach the cluster

```bash
helm list -A
```

![Figure 4 — helm list -A with headings only](evidence/4.png)
*Figure 4 — Only column headings are printed, because no releases exist yet. The command running without error proves Helm can reach the cluster.*

### Step 0.5 — Create the lab directory

```bash
mkdir -p dso202-helm-lab/environments
cd dso202-helm-lab
```

![Figure 5 — lab directory created](evidence/5.png)
*Figure 5 — Lab directory created; it will be committed to version control.*

**✅ Checkpoint:** three `Ready` nodes, Helm installed, and `helm list -A` runs without error.

---

## 4. Stage 1 — Consuming a Published Chart

**Purpose:** practise the full release lifecycle on a finished chart (podinfo 6.15.0) before writing one.

### 4.1 Working with a chart repository

#### Step 1.1 — Add the repository and update the index

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update
```

![Figure 6 — repository added and updated](evidence/6.png)
*Figure 6 — `repo add` records the URL; `repo update` downloads the index into the local cache.*

#### Step 1.2 — Search the cached index

```bash
helm search repo podinfo
helm search repo podinfo --versions | head -4
```

![Figure 7 — search results](evidence/7.png)
*Figure 7 — Available podinfo versions. **CHART VERSION** is the package version; **APP VERSION** is the version of the software inside it.*

#### Step 1.3 — Read the chart before installing

Installing a chart without reading it means running unreviewed code with my own cluster permissions.

```bash
helm show chart podinfo/podinfo --version 6.15.0
helm show values podinfo/podinfo --version 6.15.0 | head -20
```

![Figure 8 — chart metadata and default values](evidence/8.png)
*Figure 8 — Chart metadata and default values. `replicaCount` and `ui.message` are the settings overridden in the next step.*

### 4.2 Installing and inspecting a release

#### Step 1.4 — Install the release

```bash
helm install my-podinfo podinfo/podinfo \
  --version 6.15.0 \
  --namespace dso202-helm --create-namespace \
  --set replicaCount=2 \
  --set ui.message="Hello from DSO202" \
  --wait --timeout 3m
```

| Argument | Purpose |
| --- | --- |
| `my-podinfo` | release name |
| `podinfo/podinfo` | chart reference (`<repo alias>/<chart>`) |
| `--version 6.15.0` | pins the chart version so the install is repeatable |
| `--namespace … --create-namespace` | target namespace, created if missing |
| `--set` | overrides individual values |
| `--wait --timeout 3m` | waits for objects to become ready before reporting success |

![Figure 9 — install output](evidence/9.png)
*Figure 9 — Release installed: `STATUS: deployed`, `REVISION: 1`.*

#### Step 1.5 — List releases

```bash
helm list -n dso202-helm
helm list
```

![Figure 10 — helm list with and without -n](evidence/10.png)
*Figure 10 — The second command prints only headings: releases are namespaced and the current namespace is `default`.*

#### Step 1.6 — Confirm the Kubernetes objects

```bash
kubectl get deploy,svc,pods -n dso202-helm
```

![Figure 11 — Deployment, Service and Pods](evidence/11.png)
*Figure 11 — Deployment `2/2` ready, a ClusterIP Service, and two running Pods (Pod suffixes are random).*

#### Step 1.7 — Reach the application with port-forwarding

```bash
kubectl -n dso202-helm port-forward deploy/my-podinfo 8080:9898
```

![Figure 12 — application reached through port-forward](evidence/12.png)
*Figure 12 — The application is reachable on `localhost:8080` and shows the custom message "Hello from DSO202".*

#### Step 1.8 — Inspect the release record

Each `helm get` subcommand reads the stored release record, not the live objects.

```bash
helm status my-podinfo -n dso202-helm
helm get values my-podinfo -n dso202-helm
helm get values my-podinfo -n dso202-helm --all | head -5
helm get manifest my-podinfo -n dso202-helm | grep -E '^(# Source|kind:)'
```

| Command | Shows |
| --- | --- |
| `helm status` | release summary and its live resources |
| `helm get values` | only user-supplied values |
| `helm get values --all` | merged values (defaults + user values) |
| `helm get manifest` | the exact YAML sent to the API server |

![Figure 13 — status, values and manifest](evidence/13.png)
*Figure 13 — Release status, user-supplied values, computed values and the rendered object kinds.*

#### Step 1.9 — Examine where Helm stores the release

```bash
kubectl get secrets -n dso202-helm --show-labels
```

![Figure 14 — release Secret with labels](evidence/14.png)
*Figure 14 — Each revision is stored as a Secret named `sh.helm.release.v1.<release>.v<revision>`. Its labels (`owner=helm`, `name`, `status`, `version`) let the history be found with a label selector.*

> **Note:** because values are stored in this Secret, anyone who can read Secrets in the namespace can recover them — Helm does not keep secret values secret.

### 4.3 Upgrading — a failure demonstration

#### Step 1.10 — Upgrade with only the new value (deliberately incomplete)

```bash
helm upgrade my-podinfo podinfo/podinfo --version 6.15.0 -n dso202-helm \
  --set ui.color="#2e7d32"
helm get values my-podinfo -n dso202-helm
kubectl get deploy my-podinfo -n dso202-helm -o jsonpath='{.spec.replicas}{"\n"}'
```

![Figure 15 — upgrade loses earlier values](evidence/15.png)
*Figure 15 — The upgrade succeeded, but replicas dropped from 2 to 1 and the custom message disappeared.*

**Observation:** when `helm upgrade` receives *any* value flag, it starts again from the chart defaults and applies only the values in that command. The values from the install were discarded silently.

#### Step 1.11 — Upgrade correctly with `--reuse-values`

```bash
helm upgrade my-podinfo podinfo/podinfo --version 6.15.0 -n dso202-helm \
  --reuse-values --set replicaCount=2 --set ui.message="Hello from DSO202"
helm get values my-podinfo -n dso202-helm
```

![Figure 16 — values merged with --reuse-values](evidence/16.png)
*Figure 16 — Previous values are kept and merged with the new ones (colour, replicas and message are all present).*

| Flag | Behaviour |
| --- | --- |
| `--reset-values` | chart defaults + this command's values only |
| `--reuse-values` | previous values + this command's values |
| `--reset-then-reuse-values` | new chart defaults → previous values → this command's values (safest when changing chart version) |

### 4.4 History and rollback

#### Step 1.12 — Roll back to revision 1

```bash
helm rollback my-podinfo 1 -n dso202-helm
helm history my-podinfo -n dso202-helm
helm get values my-podinfo -n dso202-helm
```

![Figure 17 — rollback and history](evidence/17.png)
*Figure 17 — The rollback created **revision 4** ("Rollback to 1"); revisions 2 and 3 remain as `superseded`.*

**Observation:** a rollback never rewrites history. It creates a new revision that copies an earlier one, so the full sequence of changes stays auditable.

### 4.5 Uninstalling

#### Step 1.13 — Remove the release

```bash
helm uninstall my-podinfo -n dso202-helm
kubectl get all,secrets -n dso202-helm
kubectl get namespace dso202-helm
```

![Figure 18 — release uninstalled](evidence/18.png)
*Figure 18 — All objects and release Secrets are gone, but the namespace remains, because `--create-namespace` does not make the namespace part of the release.*

**✅ Checkpoint:** repository added, chart read, release installed, inspected, upgraded (including the values-loss failure), rolled back and uninstalled.

---

## 5. Stage 2 — Chart Structure and Chart.yaml

**Purpose:** understand the fixed layout of a chart and create the skeleton of my own `webapp` chart.

### Step 2.1 — Generate a reference scaffold

```bash
helm create scaffold
find scaffold -type f | sort
```

![Figure 19 — scaffold files](evidence/19.png)
*Figure 19 — Files generated by `helm create`. The Helm 4 scaffold includes `httproute.yaml` (Gateway API) alongside `ingress.yaml`.*

The scaffold was used only as a reference and then removed (`rm -rf scaffold`); the `webapp` chart was written file by file so every line is understood.

### Step 2.2 — Build the webapp chart skeleton

```bash
mkdir -p webapp/templates/tests webapp/charts
```

Then the following files were created:

| File | Role |
| --- | --- |
| `webapp/Chart.yaml` | chart identity: `apiVersion: v2`, `name: webapp`, `version: 0.1.0`, `appVersion: "1.30-alpine"`, `kubeVersion: ">=1.30.0-0"` |
| `webapp/values.yaml` | documented default configuration (replicas, image, page content, service, resources, annotations, `global`) |
| `webapp/.helmignore` | files excluded when packaging (`.git/`, editor/backup files) |

![Figure 20 — webapp chart skeleton](evidence/20.png)
*Figure 20 — The `webapp` chart skeleton.*

**Key point — `version` vs `appVersion`:** `version` is the version of the *chart* and must increase whenever any chart file changes; `appVersion` is the version of the *application* inside (here the nginx image tag). It is quoted so YAML keeps it as a string.

---

## 6. Stage 3 — Writing the webapp Templates

**Purpose:** implement the chart using the conventions found in industry charts.

### Step 3.1 — Templates created

| File | What it does |
| --- | --- |
| `templates/_helpers.tpl` | named templates: `webapp.name`, `webapp.fullname` (truncated to 63 chars), `webapp.chart`, `webapp.selectorLabels` (stable), `webapp.labels` (full recommended label set), `webapp.environment` (uses `required`) |
| `templates/configmap.yaml` | the HTML page, built entirely from values and built-in objects |
| `templates/deployment.yaml` | nginx Deployment with probes, resources, a `checksum/config` annotation and the image tag defaulting to `appVersion` |
| `templates/service.yaml` | Service; `nodePort` is emitted only when the type is `NodePort` and a port is set |
| `templates/NOTES.txt` | post-install instructions for reaching the application |
| `templates/tests/test-connection.yaml` | `helm test` Pod that fetches the page and checks the environment name |

### Step 3.2 — Environment values files

These live **outside** the chart because they belong to the deployment of the chart, not the chart itself.

`environments/dev.yaml`

```yaml
# Overrides for the development release.
replicaCount: 1
page:
  environment: dev
  message: "Development build - not for customers"
service:
  type: NodePort
  nodePort: 30080
```

`environments/prod.yaml`

```yaml
# Overrides for the production release.
replicaCount: 3
page:
  environment: prod
  message: "Production"
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 128Mi
```

### Step 3.3 — Render the chart locally

`helm template` performs the rendering steps of `helm install` without contacting the cluster.

```bash
helm template webapp-dev ./webapp -f environments/dev.yaml --namespace dso202-dev
```

![Figure 21 — rendered manifests for dev](evidence/21.png)
*Figure 21 — Rendered ConfigMap, Service, Deployment and test Pod for the dev environment.*

**Observations:**

1. Objects are output in kind order (ConfigMap → Service → Deployment), not file order, so the ConfigMap exists before the Deployment mounts it.
2. Object names are just `webapp-dev`, because the release name already contains the chart name.
3. The image resolved to `nginx:1.30-alpine` because `image.tag` is empty and falls back to `appVersion`.
4. The `checksum/config` annotation is a SHA-256 of the rendered ConfigMap; changing page content changes the hash and triggers a rolling update.

**✅ Checkpoint:** the full render completes without error.

---

## 7. Stage 4 — Values: Sources, Precedence and Merging

**Purpose:** prove how Helm merges values from several sources.

**Precedence (lowest → highest):** chart `values.yaml` → `-f` files (rightmost wins) → `--set` flags.
**Merge rules:** maps merge deeply; lists and scalars are replaced; `null` deletes a key.

### Step 4.1 — `--set` overrides a values file

```bash
helm template webapp-dev ./webapp -f environments/dev.yaml --set replicaCount=2 \
  -s templates/deployment.yaml | grep 'replicas:'
```

![Figure 22 — replicas: 2](evidence/22.png)
*Figure 22 — `--set replicaCount=2` overrides `replicaCount: 1` from `dev.yaml`.*

### Step 4.2 — Layering environment files (deep merge)

**dev then prod:**

```bash
helm template webapp-dev ./webapp -f environments/dev.yaml -f environments/prod.yaml \
  -s templates/service.yaml | grep -E 'type:|nodePort'
helm template webapp-dev ./webapp -f environments/dev.yaml -f environments/prod.yaml \
  -s templates/deployment.yaml | grep -E 'replicas:|environment'
```

![Figure 23 — dev then prod](evidence/23.png)
*Figure 23 — The result is "prod" with 3 replicas, but it still exposes `NodePort 30080` from dev, because `prod.yaml` never mentions `service`.*

**prod then dev:**

```bash
helm template webapp-dev ./webapp -f environments/prod.yaml -f environments/dev.yaml \
  -s templates/deployment.yaml | grep -E 'replicas:|environment|cpu'
```

![Figure 24 — prod then dev](evidence/24.png)
*Figure 24 — The result is "dev" with 1 replica, but it keeps production CPU settings (`500m` / `100m`), because `dev.yaml` never mentions `resources`.*

**Observation:** layering environment files leaks settings between environments. Each environment file should be applied alone on top of the chart defaults.

### Step 4.3 — `--set` type conversion

Annotation values must be strings.

```bash
helm template webapp-dev ./webapp -f environments/dev.yaml -s templates/deployment.yaml \
  --set 'podAnnotations.prometheus\.io/scrape=true' | grep scrape
helm template webapp-dev ./webapp -f environments/dev.yaml -s templates/deployment.yaml \
  --set-string 'podAnnotations.prometheus\.io/scrape=true' | grep scrape
```

![Figure 25 — boolean vs string annotation](evidence/25.png)
*Figure 25 — `--set` produces the boolean `true` (rejected by the API server at install time); `--set-string` produces the string `"true"` (valid).*

### Step 4.4 — Delete a default with `null`

```bash
helm template webapp-dev ./webapp -f environments/dev.yaml -s templates/deployment.yaml \
  --set resources.limits=null | sed -n '/resources:/,/volumeMounts/p'
```

![Figure 26 — limits removed](evidence/26.png)
*Figure 26 — `resources.limits` is removed entirely; only `requests` remain.*

---

## 8. Stage 5 — Validating Charts Before Installation

**Purpose:** catch errors before anything reaches the cluster, and see that no single tool catches everything.

### Failure 1 — Missing required value

```bash
helm template webapp-dev ./webapp
```

![Figure 27 — required value error](evidence/27.png)
*Figure 27 — Rendering stops with the chart's own message: `page.environment must be set (dev, staging or prod)`.*

The same chart was then linted without values:

```bash
helm lint ./webapp; echo "exit code: $?"
helm lint --strict ./webapp; echo "exit code: $?"
```

![Figure 28 — helm lint exit code 0](evidence/28.png)
*Figure 28 — `helm lint` only **warns** about the missing value and exits with code `0`, even with `--strict`.*

**Observation:** a CI pipeline relying on `helm lint` alone would pass a chart that cannot render. Both `helm lint` **and** `helm template` must be run for every environment.

### Final check — Every environment passes

```bash
for env in dev prod; do
  helm lint ./webapp -f environments/$env.yaml &&
  helm template webapp-$env ./webapp -f environments/$env.yaml > /dev/null &&
  echo "$env: ok"
done
```

![Figure 29 — dev: ok, prod: ok](evidence/29.png)
*Figure 29 — The chart lints and renders cleanly for both environments. The `[INFO] icon is recommended` line is advisory only.*

### What each check covers

| Check | Missing required value | Wrong value type | Bad indentation | Unknown K8s field | Needs cluster |
| --- | --- | --- | --- | --- | --- |
| `helm lint` | warning only | yes (with schema) | yes | no | no |
| `helm template` | yes | yes (with schema) | yes | no | no |
| `helm install --dry-run=server` | yes | yes (with schema) | yes | no | yes |
| `helm template \| kubectl apply --dry-run=server -f -` | yes | yes (with schema) | yes | yes | yes |

**✅ Checkpoint:** the validation loop prints `ok` for both `dev` and `prod`.

---

## 9. Summary of Results

| Stage | Outcome |
| --- | --- |
| 0 | kind cluster running; Helm 4 installed and connected |
| 1 | podinfo installed, inspected, upgraded, rolled back (revision 4) and uninstalled |
| 2 | `webapp` chart skeleton created with `Chart.yaml`, `values.yaml`, `.helmignore` |
| 3 | six templates written; chart renders cleanly for dev |
| 4 | value precedence, deep merging, type conversion and `null` deletion demonstrated |
| 5 | `required` enforced; lint weakness shown; both environments validate successfully |

### Key lessons

- **Always pin `--version`** — otherwise the same command installs different software on different days.
- **Upgrades with partial `--set` flags silently drop earlier values.** Keep all values in version-controlled files and pass them on every upgrade.
- **Rollback appends a revision**; it does not erase history and does not restore data.
- **Release records hold every value**, including secrets, inside the cluster.
- **Environment files must not be layered**; settings leak between environments.
- **`helm lint` alone is not enough** — render every environment in CI and fail on any non-zero exit code.

---

## 10. Conclusion

This practical covered the full Helm workflow: consuming a published chart and managing its lifecycle, then authoring, configuring and validating a chart of my own. The failure demonstrations — lost values on upgrade, leaking settings between environments, invalid `--set` types and `helm lint`'s missed error — showed why industry practice relies on pinned versions, per-environment values files kept in Git, and multiple layers of validation before any release reaches a cluster.