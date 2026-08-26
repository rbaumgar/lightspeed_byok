---
title: "ACME Deployment SOP"
url: "https://docs.acme.example/openshift/deployment-sop"
---

# Deployment Runbook (SOP)

## Purpose

Step-by-step instructions to deploy a service to production on ACME OpenShift clusters.

## Pre-deployment

- Verify image in `quay.io/acme/` with desired tag.
- Confirm `*-prod` namespace exists and you have `oc` access.
- Run security and smoke tests in staging.

## Deployment steps

1. Scale down non-critical workloads if needed.
2. Apply manifests:

```bash
oc apply -f k8s/namespace.yaml
oc apply -f k8s/deployment.yaml
oc apply -f k8s/service.yaml
```

3. Monitor rollout:

```bash
oc rollout status deployment/<app-name> -n <app-name>-prod
```

4. Run post-deploy checks: readiness, health endpoints, and tracing.

## Rollback

- If errors occur, rollback to previous revision:

```bash
oc rollout undo deployment/<app-name> -n <app-name>-prod
```

## Notes

Keep deployment manifests immutable per-release via a GitOps workflow.
