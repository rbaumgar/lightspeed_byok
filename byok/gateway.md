---
title: "ACME Gateway API Troubleshooting"
url: "https://docs.acme.example/openshift/gateway"
---


# Gateway API Troubleshooting

## Gateway is not programmed

Check the GatewayClass:

oc get gatewayclass

Check the Gateway:

oc describe gateway <gateway-name>