Sure. A good **OpenShift Lightspeed BYO Knowledge** example is an organization-specific SOP. Red Hat documents BYO Knowledge as a way to create a RAG database from custom Markdown content, package it as a container image, push it to a registry, and reference that image from the `OLSConfig` resource. ([Red Hat Documentation][1])

### Example: Company-specific OpenShift deployment standards

Imagine your organization has these internal rules:

* Production workloads must use the `prod` namespace.
* Applications must have CPU/memory requests and limits.
* Images must come from the company's Quay registry.
* Production deployments require 3 replicas.
* Routes must use TLS.
* The organization uses a specific StorageClass.

You could create a Markdown document such as [demo](/byok/general.md)

### What this enables

After adding this knowledge to Lightspeed, an administrator could ask:

> **"What are ACME's requirements for deploying an application to production?"**

Instead of relying only on the general OpenShift knowledge, Lightspeed can retrieve the organization's custom guidance and answer along the lines of:

> ACME requires production applications to use a `*-prod` namespace, images from `quay.io/acme/`, resource requests and limits, at least 3 replicas, TLS for externally exposed Routes, and the `acme-prod` StorageClass for persistent storage.

This is the key value of BYO Knowledge: **you can teach OpenShift Lightspeed organization-specific procedures and standards without modifying the underlying OpenShift documentation.** ([Red Hat Documentation][1])

For the actual implementation, Red Hat's documented flow is **Markdown → BYO Knowledge tool → RAG container image → registry → `OLSConfig` configuration**. ([Red Hat Documentation][2])

The important distinction is that **BYO Knowledge isn't a toggle you simply enable**. The procedure is:

**Markdown documents → BYO Knowledge tool → RAG container image → registry → `OLSConfig.spec.ols.rag`**

The current Red Hat documentation describes BYO Knowledge as a **Technology Preview** feature. ([Red Hat Documentation][3])

## Actual procedure

### 1. Prepare your knowledge

Create one or more `.md` files containing your organization's knowledge.

For example:

```text
byok/
├── production-standards.md
├── troubleshooting.md
└── deployment-sop.md
```

The tool currently accepts **Markdown (`.md`) files**. You can optionally put `title` and `url` metadata in the Markdown front matter so Lightspeed can show the source information. ([Red Hat Documentation][3])

Example:

```markdown
---
title: "ACME Production Deployment Standards"
url: "https://docs.acme.example/openshift/production"
---

# ACME Production Deployment Standards

Production applications must run with a minimum of three replicas.

All production container images must come from:

quay.io/acme/

...
```

### 2. Make sure the prerequisites are available

You need:

* OpenShift Lightspeed Operator installed.
* An LLM provider configured for Lightspeed.
* `podman` installed.
* Authentication to `registry.redhat.io`.
* A container registry where you can push the resulting image, such as Quay.
* Permission to modify the cluster-scoped `OLSConfig` resource. ([Red Hat Documentation][3])

Log into the Red Hat registry:

```bash
podman login registry.redhat.io
```

### 3. Run the BYO Knowledge tool

This is the part that actually **creates the RAG database**.

Suppose your Markdown is in:

```text
byok
```

and you want the generated the Vector.db in:

```text
output
```

Run:

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

The tool processes your Markdown and creates the Vector.db files.

```bash
Arguments used: Namespace(input_dir='/markdown', embedding_model_dir='embeddings_model', embedding_model_name='sentence-transformers/all-mpnet-base-v2', chunk_size=380, chunk_overlap=0, output_dir='/output', index_id='vector_db_index')
2026-08-19 13:40:27,747 - INFO - Loading SentenceTransformer model from embeddings_model.
LLM is explicitly disabled. Using MockLLM.
file_path: /markdown/gateway.md, title: ACME Gateway API Troubleshooting, docs_url: https://docs.acme.example/openshift/gateway
file_path: /markdown/general.md, title: ACME Production Deployment Standards, docs_url: https://docs.acme.example/openshift/production

$ ls -l output
total 36
-rw-r--r--. 1 demo demo 9261 19. Aug 15:40 default__vector_store.json
-rw-r--r--. 1 demo demo 7739 19. Aug 15:40 docstore.json
-rw-r--r--. 1 demo demo   18 19. Aug 15:40 graph_store.json
-rw-r--r--. 1 demo demo   72 19. Aug 15:40 image__vector_store.json
-rw-r--r--. 1 demo demo  352 19. Aug 15:40 index_store.json
-rw-r--r--. 1 demo demo  267 19. Aug 15:40 metadata.json
```

### 4. Generate the RAG image and push it

The tool produces an image TAR in your output directory.

```shell
echo -e 'FROM registry.access.redhat.com/ubi9/ubi-minimal\nCOPY . /rag/vector_db' | \
  podman build -t quay.io/rbaumgar/my-byok-image:latest -f - $MYDIR/output/
podman push quay.io/rbaumgar/my-byok-image:latest
```

At this point your custom knowledge RAG is available as a container image. ([Red Hat Documentation][3])

## 6. Tell OpenShift Lightspeed to use the image

This is the **actual Lightspeed configuration step**.

You modify the `OLSConfig` resource and add:

```yaml
apiVersion: ols.openshift.io/v1alpha1
kind: OLSConfig
metadata:
  name: cluster
spec:
  ols:
    rag:
      - image: quay.io/$QUAY_USER/openshift-lightspeed-knowledge:latest
```

You can do this from:

**OpenShift Console → Operators → Installed Operators → OpenShift Lightspeed Operator → OLSConfig → `cluster` → YAML**

Add the `rag` section under `spec.ols`, then **Save**. ([Red Hat Documentation][3])

Or, if you have the existing OLSConfig in a file:

```bash
oc apply -f OLSConfig.yaml
```

### 7. Private registry?

If the image isn't accessible using the cluster's normal pull secret, configure `imagePullSecrets`:

```yaml
spec:
  ols:
    imagePullSecrets:
      - name: my-registry-secret

    rag:
      - image: quay.io/$QUAY_USER/openshift-lightspeed-knowledge:latest
```

Red Hat specifies that these pull secrets are used for the BYO Knowledge RAG images. ([Red Hat Documentation][3])

## 8. Verify it

Open the OpenShift Lightspeed assistant and ask something that **only exists in your custom Markdown**.

For example, if your document says:

> Production deployments require a minimum of three replicas.

Ask:

> **"According to our organization's production standards, how many replicas are required?"**

Lightspeed should retrieve the information from your BYO Knowledge RAG database and answer based on that content. ([Red Hat Documentation][3])

## If you want Lightspeed to use ONLY your knowledge

By default, you're **adding** your RAG alongside the normal OpenShift documentation.

If you specifically want:

> "Don't use the built-in OpenShift documentation; answer using only my BYO Knowledge databases."

set:

```yaml
spec:
  ols:
    byokRAGOnly: true
    rag:
      - image: quay.io/$QUAY_USER/openshift-lightspeed-knowledge:latest
```

Red Hat documents `byokRAGOnly: true` for disabling the default OpenShift documentation RAG. ([Red Hat Documentation][3])

### The complete picture

```text
                  Your documentation
                         │
                         ▼
                 *.md Markdown files
                         │
                         ▼
              ┌─────────────────────┐
              │   BYO Knowledge     │
              │       tool          │
              └──────────┬──────────┘
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
                  LLM generates
                     answer
```

One important caveat: **BYO Knowledge is currently documented by Red Hat as Technology Preview**, so it isn't recommended for production use under the normal production SLA. ([Red Hat Documentation][3])


[Red Hat OpenShift Lightspeed — BYO Knowledge documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html-single/configure/index)

[1]: https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html/configure/ols-configuring-openshift-lightspeed "Chapter 1. Configuring and deploying OpenShift Lightspeed | Configure | Red Hat OpenShift Lightspeed | 1.0 | Red Hat Documentation"

[2]: https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html-single/configure/index "Configure | Red Hat OpenShift Lightspeed | 1.0 | Red Hat Documentation"

[3]: https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html/configure/ols-configuring-openshift-lightspeed "Chapter 1. Configuring and deploying OpenShift Lightspeed | Configure | Red Hat OpenShift Lightspeed | 1.0 | Red Hat Documentation"
