---
title: "ACME Production Deployment Standards"
url: "https://docs.acme.example/openshift/production"
---

# ACME Production Deployment Standards

## Purpose

This document defines the standard deployment requirements for applications
running on ACME's OpenShift clusters.

## Production namespaces

All production applications must be deployed into a namespace following
the naming convention:

    <application>-prod

For example:

    payments-prod
    customer-api-prod

Do not deploy production workloads into the `default` namespace.

## Container images

Production workloads must use images from the ACME Quay registry:

    quay.io/acme/

Do not use images directly from Docker Hub in production.

Example:

    quay.io/acme/payments:1.4.2

## Resource requirements

Every production Deployment must define CPU and memory requests and limits.

Example:

    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "1"
        memory: "512Mi"

## Replicas

Production applications must run with a minimum of 3 replicas.

Example:

    replicas: 3

## Network security

All production applications exposed outside the cluster must use an
OpenShift Route with TLS enabled.

Use edge TLS termination unless the application requires another
termination strategy.

## Storage

Persistent applications must use the `acme-prod` StorageClass.

Example:

    storageClassName: acme-prod

## Deployment checklist

Before deploying an application to production, verify:

1. The application is deployed to the correct `*-prod` namespace.
2. Container images come from `quay.io/acme/`.
3. CPU and memory requests and limits are defined.
4. At least 3 replicas are configured.
5. External Routes use TLS.
6. Persistent volumes use the `acme-prod` StorageClass.