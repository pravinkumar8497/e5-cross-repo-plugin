---
service: e5-platform-ss-credential-monitor
feature: credential-release-appname
status: active
related_files:
  - KubernetesPodsMonitor.java
  - MonitoringClientImpl.java (addCredentialFetchEventToStore)
integrations: []
---

# credential-release-appname

Spec: `e5-feature-specs/2026-09-06-credential-release-appname/design.md`.

## 2026-09-06 — credential event flow

`CredentialFetchedEvent`/`CredentialReleasedEvent` (consumed from Kafka,
published by `e5-rf-fb-worker-core`) currently carry only `pod_name` and
`lock_id` (fetched) / `pod_name` and `lock_id` (released) — no app name.
`MonitoringClientImpl` forwards these into `e5-deployment-mgmt-service` via
`addAssignedCredentialToStore` / `updateReleaseStatusToStore`
(`PodCredentialAssignment` / `StoreCredentialReleaseRequest` protos).

`CredentialFetchedEvent` gains an optional `app_name` field, passed through
to `PodCredentialAssignment.appName` when present.

## 2026-09-06 — main branch was stale; implemented on release/v1.1.0

`main` was missing `resource_name`/`kube_namespace` fields on
`CredentialFetchedEvent` and their pass-through in
`MonitoringClientImpl.addCredentialFetchEventToStore` that `release/v1.1.0`
already had (that wiring is NOT missing in production — it was just missing
from the stale `main` checkout this repo folder had checked out).

Implemented on branch `defect/credential-release-appname` off
`release/v1.1.0` (commit 225d373): `app_name` added to `mgmt.proto`'s
`PodCredentialAssignment` (field 7, optional) and threaded through
identically to the existing `resourceName` (field 6) pattern.

**Local build limitation:** this repo also needs `e5-task-sdk-java` from the
private Artifactory (401 without credentials) — could not run `./gradlew
compileJava` in this environment; verified by manual diff review against the
resourceName pattern it mirrors exactly.
