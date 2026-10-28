---
integration: e5-platform-ss-interface <-> e5-workflow-manager-service
protocol: kafka (task-manager/AKST routing, not a direct call)
direction: e5-workflow-manager-service is the eventual consumer; no direct dependency edge between them
---

# e5-platform-ss-interface ↔ e5-workflow-manager-service

## What crosses the boundary

Not a direct call — the interface never talks to workflow-manager
directly. The path is:

```
ScaleAppWorkerServiceImpl → FbWorkerPodScalingService
  builds a TaskTransport (AKST v2 protocol envelope):
    fbName: FB_WORKER_POD_SCALE_APP_WORKER
    fbPayload: { ...mode-specific, see below... }
    destinationType: akst, destinationConfig.vqName = INTERFACE_TASK_WORKFLOWMANAGER_OUTBOUND_QUEUE
  → performTask (DMS, records + returns taskId)
  → Kafka produce to TASK_MANAGER_INBOUND_TOPIC
      (task-manager service, not in this set, routes onward via the vqName
       label to workflow-manager's own AKST inbound listener)
```

## The payload contract — half-implemented as of 2026-10-26

**Per-app mode** (pre-existing, both sides agree):
```json
{ "deploymentVersionId": "...", "appName": "...", "minReplicas": N, "maxReplicas": N }
```
Read on the workflow-manager side as `FbScaleAppWorkerDTO(appName, deploymentVersionId, minReplicas, maxReplicas)`.

**Group mode** (new, interface side only — `e5-platform-ss-interface`'s
`scale-app-worker-group-support` feature):
```json
{ "groupName": "...", "groupCount": N }
```
Sent under the same `fbName: FB_WORKER_POD_SCALE_APP_WORKER`. **Nothing on
the workflow-manager side has been confirmed to consume this shape** — the
core scaling behavior for a group was explicitly out of scope for the
interface task ("another team member... working on its core functionality
in workflow-manager"). Unverified/undecided as of this note:

- Whether `FbScaleAppWorkerDTO` (or a new DTO) needs a `groupName`/`groupCount`
  variant, or a separate task type entirely.
- Whether `groupCount` changes the group's actual credential count
  (`shared_credential_group.total_count` in DMS) or just the running replica
  count per member — this decides whether a resize needs to go through DMS's
  capacity rule, and whether the next tenant-config deployment would
  silently overwrite it.
- Whether workflow-manager fans the group out to its members itself (one
  task in, N actions out) or expects the interface to send one task per
  member (it currently sends one task for the whole group).

**Before relying on group-mode `scaleAppWorker` actually scaling anything in
production: confirm with whoever implements the workflow-manager side that
the payload shape above is what they're building against** — this doc
records what the interface emits, not an agreed contract.
