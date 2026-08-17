Yes. For your **OpenShift 4.22 + OpenShift Lightspeed + Gateway API** setup, I'd build it as a proper Tekton pipeline with the **OpenShift internal registry** as the destination.

One correction to my previous answer: I would **not put the Lightspeed RAG tool itself directly into a Tekton step and assume it behaves like a normal build image**. Red Hat's documented tool invocation runs the RAG tool container with `/dev/fuse`, mounts `/markdown` and `/output`, and produces a `.tar` containing the generated RAG image. ([Red Hat Documentation][1])

So the robust architecture is:

```text
                         Git
                          │
                          │ Markdown
                          ▼
                  ┌───────────────┐
                  │ Tekton        │
                  │ Pipeline      │
                  └───────┬───────┘
                          │
             ┌────────────┴────────────┐
             │                         │
             ▼                         ▼
        git-clone                 validate
             │
             ▼
      BYOK builder pod
             │
             │ podman run
             ▼
 lightspeed-rag-tool-rhel9
             │
             │ RAG image .tar
             ▼
          podman load
             │
             ▼
    OpenShift internal registry
             │
             │ ImageStream
             ▼
      OpenShift Lightspeed
             │
             ▼
             RAG
```

Red Hat documents that Lightspeed can automatically detect changes to a floating BYOK image tag through an OpenShift `ImageStream`; the cluster checks ImageStreams every 15 minutes. ([Red Hat Documentation][2])

---

# 1. Recommended namespaces

I'd separate the knowledge build from the Lightspeed namespace:

```text
openshift-lightspeed
        │
        └── Lightspeed Operator / OLSConfig

lightspeed-knowledge
        │
        ├── Tekton Pipeline
        ├── ServiceAccount
        ├── Secrets
        └── build PVC
```

Create the namespace:

```bash
oc new-project lightspeed-knowledge
```

---

# 2. Git repository

I'd structure your repository something like:

```text
openshift-knowledge/
├── README.md
│
├── gateway-api/
│   ├── architecture.md
│   ├── gatewayclass.md
│   ├── gateway.md
│   ├── httproute.md
│   └── troubleshooting.md
│
├── networking/
│   ├── dns.md
│   ├── ingress.md
│   └── load-balancing.md
│
├── monitoring/
│   ├── prometheus.md
│   ├── alerts.md
│   └── dashboards.md
│
├── disaster-recovery/
│   ├── architecture.md
│   ├── backup.md
│   └── restore.md
│
└── runbooks/
    ├── gateway-not-working.md
    ├── application-not-reachable.md
    └── node-not-ready.md
```

Only the Markdown files are consumed by the BYO Knowledge tool; Red Hat currently documents `.md` as the supported content format. ([Red Hat Documentation][1])


# 3. Give Tekton access to Red Hat registry

The builder needs to pull:

```text
registry.redhat.io/openshift-lightspeed-tech-preview/lightspeed-rag-tool-rhel9:latest
````

Red Hat's documented process requires authentication to `registry.redhat.io`. ([Red Hat Documentation][1])

Create a pull secret from your Red Hat registry credentials:

```bash
oc create secret docker-registry redhat-registry \
  --docker-server=registry.redhat.io \
  --docker-username='<RH_USERNAME>' \
  --docker-password='<RH_PASSWORD>' \
  -n lightspeed-knowledge
```

Then attach it to the ServiceAccount used by Tekton:

```bash
oc patch serviceaccount pipeline \
  -n lightspeed-knowledge \
  -p '{"secrets":[{"name":"redhat-registry"}]}'
```

Depending on your Tekton installation, you may instead want to attach the secret as an `imagePullSecrets` entry. I'd verify the generated Pod before relying on the above.

---

# 4. ServiceAccount

Create a dedicated account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: lightspeed-byok-builder
  namespace: lightspeed-knowledge
```

We'll use this rather than the default `pipeline` account.

---

# 5. The important part: `/dev/fuse`

The Red Hat command is:

```bash
podman run -it --rm --device=/dev/fuse \
  ...
  registry.redhat.io/openshift-lightspeed-tech-preview/lightspeed-rag-tool-rhel9:latest
```

([Red Hat Documentation][1])

Therefore the Tekton Pod needs permission to access `/dev/fuse`.

For a dedicated builder ServiceAccount, I'd create a narrowly scoped SCC rather than changing the default `anyuid`/`privileged` SCC.

For example:

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: lightspeed-byok-builder
allowPrivilegedContainer: true
allowHostDirVolumePlugin: false
allowHostNetwork: false
allowHostPID: false
allowHostIPC: false
readOnlyRootFilesystem: false

allowedCapabilities:
  - SETUID
  - SETGID

runAsUser:
  type: RunAsAny

seLinuxContext:
  type: MustRunAs

fsGroup:
  type: RunAsAny

supplementalGroups:
  type: RunAsAny

volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret

users:
  - system:serviceaccount:lightspeed-knowledge:lightspeed-byok-builder
```

Then:

```bash
oc create -f scc.yaml
```

However, **I would test the exact SCC requirements against the current RAG tool image before putting this into production**, because the requirements can change with the Lightspeed release.

---

# 6. Tekton Task: run the BYOK tool

The cleanest approach is a dedicated Task that has Podman available.

For example:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: build-byok
  namespace: lightspeed-knowledge
spec:
  workspaces:
    - name: source
    - name: output

  steps:
    - name: build
      image: registry.access.redhat.com/ubi9/podman:latest

      securityContext:
        privileged: true

      env:
        - name: STORAGE_DRIVER
          value: vfs

      script: |
        #!/usr/bin/env bash
        set -euo pipefail

        echo "======================================"
        echo "OpenShift Lightspeed BYOK build"
        echo "======================================"

        echo
        echo "Markdown files:"
        find "$(workspaces.source.path)" \
          -type f \
          -name '*.md' \
          -print

        echo
        echo "Starting BYOK builder..."

        mkdir -p "$(workspaces.output.path)"

        podman run \
          --rm \
          --device=/dev/fuse \
          -v "$(workspaces.source.path)/byok:/markdown:Z" \
          -v "$(workspaces.output.path):/output:Z" \
          registry.redhat.io/openshift-lightspeed-tech-preview/lightspeed-rag-tool-rhel9:latest

        echo
        echo "Generated artifacts:"
        ls -lah "$(workspaces.output.path)"
```

The important thing is that the **inner** container is the Red Hat BYOK tool.

The directory mapping is:

```text
Tekton workspace
       │
       ├── /workspace/source
       │
       │       │
       │       └────> /markdown
       │
       └── /workspace/output
                       │
                       └────> /output
```

This follows Red Hat's documented BYOK invocation. ([Red Hat Documentation][1])

---

# 7. Load the generated image

The BYOK tool generates a tarball. Red Hat's documented next step is:

```bash
podman load < <directory>/<my-byok-image.tar>
```

([Red Hat Documentation][1])

So create a second Task:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: push-byok
  namespace: lightspeed-knowledge
spec:
  params:
    - name: image
      type: string

  workspaces:
    - name: output

  steps:
    - name: push
      image: registry.access.redhat.com/ubi9/podman:latest

      securityContext:
        privileged: true

      env:
        - name: STORAGE_DRIVER
          value: vfs

      script: |
        #!/usr/bin/env bash
        set -euo pipefail

        OUTPUT="$(workspaces.output.path)"

        echo "Looking for BYOK image..."

        TAR="$(find "$OUTPUT" -type f -name '*.tar' | head -1)"

        if [ -z "$TAR" ]; then
          echo "ERROR: No BYOK image tar found"
          exit 1
        fi

        echo "Found:"
        echo "$TAR"

        podman load < "$TAR"

        echo
        echo "Loaded images:"
        podman images

        SOURCE_IMAGE="$(podman images \
          --format '{{.Repository}}:{{.Tag}}' \
          | grep 'my-byok-image' \
          | head -1)"

        if [ -z "$SOURCE_IMAGE" ]; then
          echo "ERROR: Could not find generated BYOK image"
          exit 1
        fi

        echo "Source image: $SOURCE_IMAGE"

        podman tag \
          "$SOURCE_IMAGE" \
          "$(params.image)"

        echo "Pushing:"
        echo "$(params.image)"

        podman push \
          "$(params.image)"
```

---

# 8. Internal OpenShift registry

Now comes the useful part.

Suppose your cluster's internal registry is:

```text
image-registry.openshift-image-registry.svc:5000
```

Create an ImageStream:

```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: lightspeed-byok
  namespace: lightspeed-knowledge
```

Apply:

```bash
oc apply -f imagestream.yaml
```

Your target image becomes:

```text
image-registry.openshift-image-registry.svc:5000/lightspeed-knowledge/lightspeed-byok:latest
```

---

# 9. Let the Tekton pipeline push to the internal registry

The builder Pod must authenticate to the internal registry.

The easiest approach is normally to use the OpenShift service-account token.

For example:

```bash
oc create token lightspeed-byok-builder \
  -n lightspeed-knowledge
```

But for a production pipeline I'd avoid baking a token into YAML. Instead, create a registry authentication secret and attach it to the builder ServiceAccount.

Alternatively, because the image registry is inside the cluster, you can configure the Podman authentication dynamically in the Task.

For example:

```bash
TOKEN="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"

podman login \
  --tls-verify=false \
  -u unused \
  -p "$TOKEN" \
  image-registry.openshift-image-registry.svc:5000
```

Then:

```bash
podman push \
  --tls-verify=false \
  "$IMAGE"
```

The exact TLS setting depends on how your internal registry is exposed/configured. **Don't blindly use `--tls-verify=false` in production**; use the cluster's trusted CA where possible.

---

# 10. Complete Pipeline

Now connect the pieces:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: lightspeed-byok
  namespace: lightspeed-knowledge

spec:
  params:

    - name: git-url
      type: string

    - name: git-revision
      type: string
      default: main

    - name: image
      type: string

  workspaces:

    - name: source

    - name: output

  tasks:

    - name: clone
      taskRef:
        name: git-clone

      params:
        - name: url
          value: $(params.git-url)

        - name: revision
          value: $(params.git-revision)

      workspaces:
        - name: output
          workspace: source

    - name: build-byok
      runAfter:
        - clone

      taskRef:
        name: build-byok

      workspaces:
        - name: source
          workspace: source

        - name: output
          workspace: output

    - name: push-byok
      runAfter:
        - build-byok

      taskRef:
        name: push-byok

      params:
        - name: image
          value: $(params.image)

      workspaces:
        - name: output
          workspace: output
```

---

# 11. PipelineRun

For your environment:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: lightspeed-byok-
  namespace: lightspeed-knowledge

spec:
  taskRunTemplate:
    serviceAccountName: lightspeed-byok-builder

  pipelineRef:
    name: lightspeed-byok

  params:

    - name: git-url
      value: https://git.example.com/platform/openshift-knowledge.git

    - name: git-revision
      value: main

    - name: image
      value: image-registry.openshift-image-registry.svc:5000/lightspeed-knowledge/lightspeed-byok:latest

  workspaces:

    - name: source
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi

    - name: output
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi
```

Run it:

```bash
oc create -f pipelinerun.yaml
```

Then:

```bash
tkn pipelinerun list -n lightspeed-knowledge
```

and:

```bash
tkn pipelinerun logs \
  -f \
  -n lightspeed-knowledge \
  <PIPELINERUN>
```

---

# 12. Verify the image

After the pipeline completes:

```bash
oc get is -n lightspeed-knowledge
```

You should get something similar to:

```text
NAME              IMAGE REPOSITORY
lightspeed-byok   image-registry.openshift-image-registry.svc:5000/lightspeed-knowledge/lightspeed-byok
```

Then:

```bash
oc describe is lightspeed-byok \
  -n lightspeed-knowledge
```

You should see the `latest` tag.

You can also:

```bash
oc get istag \
  -n lightspeed-knowledge
```

---

# 13. Configure Lightspeed

Your `OLSConfig` then points to the internal registry image:

```yaml
apiVersion: ols.openshift.io/v1alpha1
kind: OLSConfig
metadata:
  name: cluster
spec:
  ols:
    rag:
      - image: image-registry.openshift-image-registry.svc:5000/lightspeed-knowledge/lightspeed-byok:latest
```

With Lightspeed 1.0.3+, `indexPath` and `indexID` are optional; their defaults are `/rag/vector_db` and `vector_db_index`. ([Red Hat Documentation][3])

---

# 14. Important: ImageStream and `latest`

This is where I would use the OpenShift-native approach rather than simply pushing an image and hoping Lightspeed notices.

Your flow becomes:

```text
Git commit
    │
    ▼
Tekton
    │
    ▼
BYOK image
    │
    ▼
ImageStream: lightspeed-byok
    │
    │ latest changes
    ▼
Lightspeed
    │
    │ checks ImageStream
    ▼
Reloads RAG
```

Red Hat explicitly documents that Lightspeed uses OpenShift `ImageStream` objects to detect changes to floating BYOK tags and checks for updates every 15 minutes. ([Red Hat Documentation][2])

So **don't use a SHA-only tag in `OLSConfig` if you want automatic updates**.

Use:

```yaml
image: image-registry.openshift-image-registry.svc:5000/lightspeed-knowledge/lightspeed-byok:latest
```

and have Tekton update `latest`.

---

# 15. Automatic build on Git push

The final piece I'd add is a **Tekton Trigger**:

```text
Git push
   │
   ▼
Webhook
   │
   ▼
EventListener
   │
   ▼
Trigger
   │
   ▼
PipelineRun
   │
   ├── clone
   ├── validate
   ├── BYOK build
   └── push :latest
             │
             ▼
       ImageStream update
             │
             ▼
       Lightspeed refresh
```

Then your developers/admins only need to:

```bash
git add .
git commit -m "Update Gateway API troubleshooting"
git push
```

No manual `podman` operation is required.

---

## One improvement I'd strongly recommend

Before the BYOK build, add a **validation Task**:

```text
validate
   │
   ├── ensure *.md exists
   ├── reject empty files
   ├── validate YAML frontmatter
   ├── check duplicate titles
   ├── check URLs
   └── optionally run markdownlint
          │
          ▼
      build-byok
```

That prevents a bad documentation commit from replacing a perfectly good RAG database.

Also, remember that Red Hat currently classifies BYO Knowledge as **Technology Preview**, so this should be treated accordingly for production use. ([Red Hat Documentation][1])

And there's an important data-flow consideration: **the documents supplied through BYO Knowledge are sent to your configured LLM provider**. For internal runbooks, credentials, secrets, customer data, etc., that needs to be considered before putting the content into the repository. ([Red Hat Documentation][2])

[Red Hat: Configuring OpenShift Lightspeed / BYO Knowledge](https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html/configure/ols-configuring-openshift-lightspeed?utm_source=chatgpt.com)

If you're using the **OpenShift internal registry**, the next thing I'd change in the example is the registry authentication/SCC section: I can give you a **single `oc apply -f byok-tekton.yaml` manifest containing the ServiceAccount, SCC, Tasks, Pipeline, ImageStream, registry auth, and Trigger**, tailored to OpenShift 4.22.

[1]: https://docs.redhat.com/ko/documentation/red_hat_openshift_lightspeed/1.0/html-single/configure/index?utm_source=chatgpt.com "Configure | Red Hat OpenShift Lightspeed | 1.0 | Red Hat Documentation"
[2]: https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html/configure/ols-configuring-openshift-lightspeed?utm_source=chatgpt.com "Chapter 1. Configuring and deploying OpenShift Lightspeed | Configure | Red Hat OpenShift Lightspeed | 1.0 | Red Hat Documentation"
[3]: https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html-single/release_notes/release_notes?utm_source=chatgpt.com "Release Notes | Red Hat OpenShift Lightspeed | 1.0 | Red Hat Documentation"
