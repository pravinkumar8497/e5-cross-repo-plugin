# e5-platform-ss-credential-monitor — notes

## 2026-09-06 — credential event flow

`KubernetesPodsMonitor` watches K8s pod events (`CoreV1Event`), only reads
`podName` from `event.getInvolvedObject().getName()` — it does not fetch
full `V1Pod` objects, so it has no access to pod labels/ownerReferences
without an extra API call.

Separately, `CredentialFetchedEvent`/`CredentialReleasedEvent` (consumed
from Kafka, published by `e5-rf-fb-worker-core`) currently carry only
`pod_name` and `lock_id` (fetched) / `pod_name` and `lock_id` (released) —
no app name. `MonitoringClientImpl` forwards these into
`e5-deployment-mgmt-service` via `addAssignedCredentialToStore` /
`updateReleaseStatusToStore` (`PodCredentialAssignment` /
`StoreCredentialReleaseRequest` protos).

Per `e5-feature-specs/2026-09-06-credential-release-appname/design.md`:
`CredentialFetchedEvent` gains an optional `app_name` field, passed through
to `PodCredentialAssignment.appName` when present.
