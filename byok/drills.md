---
title: "ACME Disaster Recovery Drill Procedure for OpenShift"
url: "https://docs.acme.example/openshift/drills"

---

# ACME Disaster Recovery Drill Procedure for OpenShift

ACME Corp requires every production OpenShift cluster to complete a disaster
recovery (DR) drill once per quarter. Drills are coordinated by the Platform
Reliability team and tracked in the ACME-DR Jira project.

## Drill requirements

- DR drills must be scheduled at least two weeks in advance through the
  ACME-DR Jira project.
- The drill must simulate the loss of one availability zone by cordoning and
  draining all nodes in that zone.
- etcd backups must be restored to the staging cluster `acme-dr-staging` and
  validated by the application teams.
- The maximum tolerated recovery time objective (RTO) at ACME Corp is
  45 minutes. Drills that exceed 45 minutes must be repeated within 30 days.

## Sign-off

After a successful drill, the Platform Reliability team lead records the
results in the ACME-DR Jira project and notifies the compliance team via the
#acme-compliance Slack channel. Compliance sign-off is mandatory before the
next production change window opens.
