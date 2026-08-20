# OpenShift Lightspeed your team's own rules with Bring Your Own Knowledge

OpenShift® Lightspeed is genuinely good at answering general OpenShift and Kubernetes questions—how a Route works, what a PodDisruptionBudget does, why your pod is stuck in `CrashLoopBackOff`. But ask it something like *"What are our requirements for deploying an app to production?"* and it has no way to know. That question isn't answered in the OpenShift docs. It's answered in your team's internal documentation, your platform team's Confluence page, or the tribal knowledge that lives in a senior engineer's head.

**Bring Your Own Knowledge (BYOK)** closes that gap. It lets you package your own Markdown documentation into a retrieval-augmented generation (RAG) database, ship it as a container image, and plug it straight into Lightspeed's `OLSConfig`. From that point on, Lightspeed can answer questions using *your* standards, not only Red Hat®'s.

It's currently a **Technology Preview** feature, so treat it as something to pilot and provide feedback on rather than something to lean on for production SLAs yet. But it's straightforward enough to try today, and the payoff—an assistant that actually knows your org's conventions—is worth the ten minutes it takes to set up.

## A concrete example

Say your organization, we'll call it ACME, has a short internal list of production rules:

- Production workloads must run in a `*-prod` namespace
- Every container must set CPU/memory requests and limits
- Images must come from `quay.io/acme/`
- Production deployments require a minimum of 3 replicas
- Externally exposed Routes must use TLS
- Persistent volumes must use the `acme-prod` StorageClass

None of that is generic OpenShift knowledge—it's ACME's own policy. Write it down as Markdown, run it through the BYO Knowledge tool, and an administrator can now ask Lightspeed:

### "What are ACME's requirements for deploying an application to production?"

and get back something like:

ACME requires production applications to run in a `*-prod` namespace, use images from `quay.io/acme/`, set resource requests and limits, run at least 3 replicas, terminate TLS on externally exposed Routes, and use the `acme-prod` StorageClass for persistent storage.

That's the whole value proposition: ** Lightspeed your standards without touching the underlying OpenShift documentation it already knows.**

## The pipeline, at a glance

BYO Knowledge isn't a check box you flip on. It's a small, linear pipeline:

```text
Markdown docs → BYO Knowledge tool → RAG container image → registry → OLSConfig.spec.ols.rag
```

Let's walk through each stage.

### 1. Write your knowledge as Markdown

The tool ingests plain `.md` files. You can optionally add `title` and `URL` front matter so Lightspeed can cite where an answer came from:

```markdown
---
title: "ACME Production Deployment Standards"
url: "https://docs.acme.example/openshift/production"
---

# ACME Production Deployment Standards

Production applications must run with a minimum of three replicas.

All production container images must come from:

quay.io/acme/
```

Organize as many files as you need:

```text
byok/
├── production-standards.md
├── troubleshooting.md
└── deployment-sop.md
```

### 2. Check your prerequisites

Before running the tool, make sure you have:

- The OpenShift Lightspeed Operator installed, with a large language model (LLM) provider already configured
- `podman` installed locally
- Authenticated access to `registry.redhat.io`
- Somewhere to push the resulting image (Quay, an internal registry, etc.)
- Permission to modify the cluster-scoped `OLSConfig` resource

```bash
podman login registry.redhat.io
```

### 3. Generate the vector database

This is where your Markdown actually becomes a RAG database. Point the tool at your input directory and an output directory:

```bash
MYDIR=`pwd`
mkdir $MYDIR/output

podman run --rm \
  -v $MYDIR/byok:/markdown:ro,Z \
  -v $MYDIR/output:/workdir/output:Z \
  --entrypoint python3.12 \
  registry.redhat.io/openshift-lightspeed-tech-preview/lightspeed-rag-tool-rhel9:latest \
  generate_embeddings_tool.py \
    -i /markdown \
    -emd embeddings_model \
    -emn sentence-transformers/all-mpnet-base-v2 \
    -o /output \
    -id vector_db_index
```

You'll see the tool chunk and embed each document:

```text
file_path: /markdown/gateway.md, title: ACME Gateway API Troubleshooting, docs_url: https://docs.acme.example/openshift/gateway
file_path: /markdown/general.md, title: ACME Production Deployment Standards, docs_url: https://docs.acme.example/openshift/production
```

and land a handful of index files in your output directory—`default__vector_store.json`, `docstore.json`, `index_store.json`, and friends.

### 4. Package it as a container image and push it

The vector store is files on disk at this point—turn it into an image with a trivial one-line Containerfile and push it wherever Lightspeed can pull from:

```bash
IMAGE=quay.io/acme/openshift-lightspeed-knowledge:latest

echo -e 'FROM registry.access.redhat.com/ubi9/ubi-minimal\nCOPY . /rag/vector_db' | \
  podman build -t $IMAGE -f - $MYDIR/output/

podman push $IMAGE
```

### 5. Point OLSConfig at it

This is the step that actually activates your knowledge base. Edit the cluster-scoped `OLSConfig`—either through **OpenShift Console → Operators → Installed Operators → OpenShift Lightspeed Operator → OLSConfig → cluster → YAML**, or by applying a manifest:

```yaml
apiVersion: ols.openshift.io/v1alpha1
kind: OLSConfig
metadata:
  name: cluster
spec:
  ols:
    rag:
      - image: quay.io/acme/openshift-lightspeed-knowledge:latest
```

```bash
oc apply -f OLSConfig.yaml
```

Lightspeed picks up the new RAG source and starts using it alongside its built-in OpenShift documentation.

### Pulling from a private registry

If the image you pushed isn't reachable with the cluster's default pull secret, tell `OLSConfig` about a dedicated one:

```yaml
spec:
  ols:
    imagePullSecrets:
      - name: my-registry-secret
    rag:
      - image: quay.io/acme/openshift-lightspeed-knowledge:latest
```

### Optional: knowledge-only mode

By default your custom RAG is additive—Lightspeed still has access to the standard OpenShift documentation too. If you'd rather it answer *exclusively* from your own knowledge base (useful for narrowly scoped internal assistants), set:

```yaml
spec:
  ols:
    byokRAGOnly: true
    rag:
      - image: quay.io/acme/openshift-lightspeed-knowledge:latest
```

## Verifying it worked

Open the Lightspeed assistant and ask something that can *only* be answered from your Markdown—not from general OpenShift knowledge. If your doc states production needs 3 replicas minimum, ask:

"According to our organization's production standards, how many replicas are required?"

If Lightspeed answers correctly, your custom knowledge is being retrieved and used.

## Automating the pipeline

Once you're happy with the manual flow, every stage described earlier—clone, generate embeddings, build, push, and patch `OLSConfig`—maps cleanly onto a CI/CD pipeline (Tekton, in an OpenShift-native setup) that runs whenever your Markdown docs change in git. The one wrinkle worth planning for: the embedding step runs `podman` inside a container, so that stage needs elevated privileges (or a FUSE device) rather than a standard restricted pipeline SCC. Everything else—pushing the image and patching `OLSConfig`—is ordinary CI plumbing with a registry secret and a ClusterRole permitting `patch` on `olsconfigs.ols.openshift.io`.

## The big picture

```text
Your documentation
        │
        ▼
  *.md Markdown files
        │
        ▼
  BYO Knowledge tool
        │
        ▼
 RAG container image
        │
        ▼
 quay.io/acme/my-knowledge:latest
        │
        ▼
     OLSConfig.rag
        │
        ▼
 OpenShift Lightspeed
        │
        ▼
 User asks a question
        │
        ▼
 RAG retrieves your docs
        │
        ▼
   LLM generates answer
```

## Wrapping up

BYO Knowledge turns OpenShift Lightspeed from a generalist into something that actually understands how *your* organization runs OpenShift. It's still Technology Preview, so don't bet production support tickets on it yet—but for internal pilots, platform teams, and anyone tired of repeating the same tribal knowledge in Slack, it's a small pipeline with an outsized payoff.

If you try it, start small: one Markdown file with your most-asked internal question, run it through the tool, and see what Lightspeed does with it.

## Documentation

[Red Hat OpenShift Lightspeed — BYO Knowledge documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html-single/configure/index#about-the-byo-knowledge-tool_ols-configuring-openshift-lightspeed)

## Status

Tested with OpenShift Lightspeed Operator 1.1.2