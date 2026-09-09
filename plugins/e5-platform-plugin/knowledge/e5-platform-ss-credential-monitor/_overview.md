---
service: e5-platform-ss-credential-monitor
role: service
depends_on: [e5-rf-fb-worker-core, e5-deployment-mgmt-service]
consumed_by: []
features: [credential-release-appname]
integrations: []
---

# e5-platform-ss-credential-monitor

Watches K8s pod events and Kafka credential-fetch/release events, forwards
credential assignment/release into `e5-deployment-mgmt-service` over gRPC.

## Features

- [credential-release-appname](./credential-release-appname.md) — forwarding
  `app_name` from the Kafka event through to `e5-deployment-mgmt-service`.

## Environment notes (repo-wide, not feature-specific)

- `KubernetesPodsMonitor` watches K8s pod events (`CoreV1Event`), only reads
  `podName` from `event.getInvolvedObject().getName()` — it does not fetch full
  `V1Pod` objects, so it has no access to pod labels/ownerReferences without an
  extra API call.
- **Branch convention:** check `git branch -a | grep release` and use the latest
  release branch as the base for new work — `main` can be stale (was missing
  fields `release/v1.1.0` already had, at time of writing).
