---
title: "ACME Troubleshooting Guide"
url: "https://docs.acme.example/openshift/troubleshooting"
---

# Troubleshooting OpenShift Issues

## Common checks

- Check pod status: `oc get pods -n <ns>`
- Inspect pod events: `oc describe pod <pod> -n <ns>`
- View logs: `oc logs <pod> [-c container] -n <ns>`

## Networking

- Verify Service and Endpoints: `oc get svc,ep -n <ns>`
- Check Routes and TLS termination: `oc get route -n <ns>`

## Persistent Storage

- Confirm PVC binding and PV health: `oc get pvc,pv -n <ns>`
- Check StorageClass and reclaim policies.

## Debugging tips

- Recreate failing pod with increased verbosity or debug image.
- Use `oc debug` to start a troubleshooting container in the same network namespace.
