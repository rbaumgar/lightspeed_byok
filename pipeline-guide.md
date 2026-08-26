# Run OpenShift Lightspeed BYOK as a Tekton pipeline

Bring Your Own Knowledge (BYOK) turns your internal Markdown into a RAG image that OpenShift Lightspeed can query. Doing that by hand — clone, embed, build, push, patch `OLSConfig` — is fine once. When docs live in git and change often, you want a pipeline instead.

This guide walks through creating and running a Tekton pipeline on OpenShift using **`kubectl apply`** for every manifest. Placeholders like `${NAMESPACE}` are filled from environment variables with `envsubst`, so you never hard-code a namespace into the YAML.

## What the pipeline does

```text
git clone → generate embeddings → build & push image → patch OLSConfig
```

| Stage | Tekton resource | What it does |
|-------|-----------------|--------------|
| 1 | `git-clone` (cluster Task) | Clones your docs repo into a shared workspace |
| 2 | `generate-embeddings` | Runs `lightspeed-rag-tool-rhel9`, builds a local vector db |
| 3 | `build-and-push` (clusterTask) | Tags and pushes the image to your registry |
| 4 | `patch-olsconfig` | Adds the new image to the cluster `OLSConfig` |

## Prerequisites

Before you start, confirm:

- OpenShift cluster with **OpenShift Pipelines** installed
- OpenShift Lightspeed Operator installed and an LLM provider configured
- `kubectl` configured for the cluster
- `envsubst` available (part of `gettext` on most Linux systems)
- A git repo containing your Markdown under a `byok/` subdirectory
- A registry service account for `registry.redhat.io` ([terms-based registry](https://access.redhat.com/terms-based-registry/))
- A push destination for the finished BYOK image (Quay, internal registry, etc.)

Clone this repository (or copy the `pipeline/` directory) and `cd` into the repo root:

```bash
git clone <this-repo-url>
cd lightspeed_byok
```

## Step 1 — Set environment variables

Export every value the manifests need. Adjust these for your environment:

```bash
# Namespace for all pipeline resources
export NAMESPACE=ols-byok

# Git repo with Markdown under docs/
export DOCS_GIT_URL=https://github.com/acme/lightspeed-docs.git
export DOCS_GIT_REVISION=main

# Image the pipeline builds and pushes
export TARGET_IMAGE=quay.io/acme/openshift-lightspeed-knowledge:latest

# Base64-encoded .dockerconfigjson for registry.redhat.io (pull)
# Generate with: cat ~/.docker/config.json | base64 -w0
export REDHAT_REGISTRY_AUTH=<base64-dockerconfigjson>

# Base64-encoded .dockerconfigjson for your push registry
export PUSH_REGISTRY_AUTH=<base64-dockerconfigjson>
```

To build the registry auth values:

```bash
# Log in once, then encode the config entry for the registry you need
podman login registry.redhat.io
export REDHAT_REGISTRY_AUTH=$(jq -r '.auths["registry.redhat.io"].auth' ~/.docker/config.json | base64 -w0)

podman login quay.io
export PUSH_REGISTRY_AUTH=$(cat ~/.docker/config.json | base64 -w0)
```

All YAML files under `pipeline/` use `${NAMESPACE}` (and related variables) as placeholders. **`envsubst` replaces them at apply time** — nothing is committed with your namespace baked in.

## Step 2 — Apply the foundation manifests

Apply resources in order. Each command pipes the templated YAML through `envsubst` and into `kubectl apply`:

```bash
envsubst < pipeline/00-namespace.yaml | kubectl apply -f -
envsubst < pipeline/01-serviceaccount.yaml | kubectl apply -f -
envsubst < pipeline/02-secrets.yaml | kubectl apply -f -
envsubst < pipeline/03-rbac.yaml | kubectl apply -f -
envsubst < pipeline/04-pvc.yaml | kubectl apply -f -
```

What each file creates:

- **`00-namespace.yaml`** — the `${NAMESPACE}` namespace
- **`01-serviceaccount.yaml`** — `ols-byok-pipeline-sa` with pull/push secret references
- **`02-secrets.yaml`** — Red Hat registry pull secret and push-registry secret
- **`03-rbac.yaml`** — privileged SCC binding plus ClusterRole/Binding to patch `OLSConfig`
- **`04-pvc.yaml`** — 2 Gi RWO PVC shared by all pipeline Tasks

Verify the namespace and ServiceAccount:

```bash
kubectl get ns "${NAMESPACE}"
kubectl get sa,secret,pvc -n "${NAMESPACE}"

kubectl patch serviceaccount pipeline -p '{"secrets": [{"name": "registry-push-secret"}]}'
```

## Step 3 — Apply the Tekton Tasks and Pipeline

Register the custom Tasks and the Pipeline that chains them together:

```bash
envsubst < pipeline/task-generate-embeddings.yaml | kubectl apply -f -
envsubst < pipeline/task-patch-olsconfig.yaml | kubectl apply -f -
envsubst < pipeline/pipeline.yaml | kubectl apply -f -
```

Confirm they exist in your namespace:

```bash
kubectl get tasks,pipeline -n "${NAMESPACE}"
```

The Pipeline's Task resolves the built-in **`git-clone`** and **`build-and-push`** Task from the `openshift-pipelines` namespace — no separate manifest is required for that step.

## Step 4 — Run the pipeline

Start a `PipelineRun`. Because the manifest uses `generateName`, each apply creates a new run:

```bash
envsubst < pipeline/pipelinerun.yaml | kubectl create -f -
```

List runs and watch progress:

```bash
kubectl get pipelinerun -n "${NAMESPACE}"
kubectl get pipelinerun -n "${NAMESPACE}" -w
```

Follow logs for the most recent run:

```bash
RUN=$(kubectl get pipelinerun -n "${NAMESPACE}" --sort-by=.metadata.creationTimestamp -o name | tail -1)
kubectl describe "${RUN}" -n "${NAMESPACE}"
kubectl logs -n "${NAMESPACE}" -l "tekton.dev/pipelineRun=${RUN##*/}" --all-containers -f
```

When the run succeeds, Lightspeed should pick up the new RAG image from `OLSConfig`. Ask a question that only your Markdown can answer to confirm.

## Step 5 — Re-run when docs change

Every time your docs repo gets new commits, export the same variables (or keep them in a shell profile) and apply a fresh `PipelineRun`:

```bash
export DOCS_GIT_REVISION=main   # or a tag/branch/commit SHA
envsubst < pipeline/pipelinerun.yaml | kubectl apply -f -
```

For automation, wire a Tekton `Trigger`/`EventListener` to your git provider so merges kick off a run without manual `kubectl apply`.

## One-shot apply helper

If you prefer a single command after exporting variables, apply everything except the `PipelineRun` in one pass:

```bash
for f in pipeline/0*.yaml pipeline/task-*.yaml pipeline/pipeline.yaml; do
  envsubst < "$f" | kubectl apply -f -
done
```

Then trigger individual runs with:

```bash
envsubst < pipeline/pipelinerun.yaml | kubectl apply -f -
```

## Troubleshooting

### `unauthorized: access to the requested resource is not authorized`

The `registry.redhat.io` pull secret is usually invalid. Regenerate it from a **registry service account**, not a personal Red Hat login, and re-apply:

```bash
envsubst < pipeline/02-secrets.yaml | kubectl apply -f -
```

### `patch-olsconfig` fails

Ensure the ClusterRoleBinding in `pipeline/03-rbac.yaml` was applied and references the same `${NAMESPACE}` you exported. The binding subject must match `ols-byok-pipeline-sa` in that namespace.

## Why automate this

Once the pipeline is in place, updating Lightspeed's knowledge is a documentation PR, not an ops checklist. Someone merges a Markdown change, a `PipelineRun` rebuilds the image and patches `OLSConfig`, and Lightspeed answers with the new content — no manual tasks required.

## Related files

| Path | Purpose |
|------|---------|
| `pipeline/00-namespace.yaml` | Namespace |
| `pipeline/01-serviceaccount.yaml` | Pipeline ServiceAccount |
| `pipeline/02-secrets.yaml` | Registry pull/push secrets |
| `pipeline/03-rbac.yaml` | SCC and OLSConfig RBAC |
| `pipeline/04-pvc.yaml` | Shared workspace PVC |
| `pipeline/task-*.yaml` | Custom Tekton Tasks |
| `pipeline/pipeline.yaml` | Pipeline definition |
| `pipeline/pipelinerun.yaml` | Runnable instance (apply per run) |

For the manual BYOK flow this pipeline automates, see [README.md](README.md).
