Sure. A good **OpenShift Lightspeed BYO Knowledge** example is an organization-specific SOP. Red Hat documents BYO Knowledge as a way to create a RAG database from custom Markdown content, package it as a container image, push it to a registry, and reference that image from the `OLSConfig` resource. ([Red Hat Documentation][1])

### Example: Company-specific OpenShift deployment standards

Imagine your organization has these internal rules:

* Production workloads must use the `prod` namespace.
* Applications must have CPU/memory requests and limits.
* Images must come from the company's Quay registry.
* Production deployments require 3 replicas.
* Routes must use TLS.
* The organization uses a specific StorageClass.

You could create a Markdown document such as [demo](/docs/general.md)

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
my-knowledge/
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
/home/user/my-knowledge
```

and you want the generated image TAR in:

```text
/home/user/byok-output
```

Run:

```bash
MYDIR=`pwd`
mkdir $MYDIR/output
podman run -it --rm --device=/dev/fuse \
  -v $XDG_RUNTIME_DIR/containers/auth.json:/run/user/0/containers/auth.json:Z \
  -v $MYDIR/my-knowledge:/markdown:Z \
  -v $MYDIR/output:/output:Z \
  registry.redhat.io/openshift-lightspeed-tech-preview/lightspeed-rag-tool-rhel9:latest
```

The tool processes your Markdown and creates a container image containing the RAG database. This is the official Red Hat procedure. ([Red Hat Documentation][3])

### 4. Load the generated image

The tool produces an image TAR in your output directory.

Load it:

```bash
podman load < $MYDIR/output/my-byok-image.tar
```

Then check:

```bash
podman images
```

You should see something similar to:

```text
REPOSITORY              TAG       IMAGE ID
localhost/my-byok-image latest    be7d1770bf10
```

### 5. Tag it for your registry

For example, using Quay:

```bash
QUAY_USER=rbaumgar
podman tag \
  localhost/my-byok-image:latest \
  quay.io/$QUAY_USER/openshift-lightspeed-knowledge:latest
```

Then push it:

```bash
podman push \
  quay.io/$QUAY_USER/openshift-lightspeed-knowledge:latest
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
