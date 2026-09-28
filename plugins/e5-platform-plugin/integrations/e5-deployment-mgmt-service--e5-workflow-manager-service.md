---
integration: e5-deployment-mgmt-service <-> e5-workflow-manager-service
protocol: db-shared-state (workflow_manager_state) + gRPC (deployment/app lookups)
direction: e5-workflow-manager-service depends on e5-deployment-mgmt-service
---

# e5-deployment-mgmt-service ↔ e5-workflow-manager-service

## What crosses the boundary

`e5-workflow-manager-service` calls into `e5-deployment-mgmt-service` for
deployment/app-name lookups (see
`knowledge/e5-workflow-manager-service/credential-release-appname.md` for the
`appName` resolution case) and for app pause/resume/terminate signal
orchestration state, tracked in `e5-deployment-mgmt-service`'s
`workflow_manager_state` table:

```
workflow_manager_state
  workflow_manager_state_id  UUID PK
  deployment_version_id       FK -> deployment_version
  reference_id, app_name, workflow_manager_action
  action_start_date_and_time
  active                       boolean
  total_signal_count, received_signal_response_count
  workflow_run_id
  timeout_in_minutes           default 65
  metadata                     jsonb
```

`active=true` means "this signal-orchestration action run is currently
in-flight" — **not** "the app is running." Don't confuse this with app
operational pause/run state, which lives in a different table entirely (see
below).

## Two different "status" concepts — do not conflate

- **App operational pause/run status**: `deployment_app_status` table
  (`app_status` column, enum `PAUSED | RUNNING | PAUSING | RESUMING |
  TERMINATED`), keyed by `deployment_id` + `application_id` (not directly by
  `deployment_version_id` — join through `deployment_version.deployment_id`).
  Queried via `DeploymentDetailsRepositoryImpl.getAppStatusByDeploymentAndAppName`
  / `getAllAppStatusesByDeployment`.
- **Signal-orchestration run state**: `workflow_manager_state` (above),
  keyed directly by `deployment_version_id` + `app_name`. Tracks an
  in-flight pause/resume/terminate *action* (how many signals sent vs.
  acknowledged, with a timeout) — this is the orchestration bookkeeping for
  *sending* a pause/resume, not the resulting app state itself.
- `workflow_ops_state` table is a third, unrelated concept: task/event
  execution status (`ACTIVE, IN_ACTIVE, COMPLETED, FAILED`), no
  `deployment_version_id` column at all.

## Diagnostic SQL

App pause/run status for a deployment version:
```sql
SELECT a.name AS app_name, das.app_status
FROM deployment_version dv
JOIN deployment_app_status das ON das.deployment_id = dv.deployment_id
JOIN application a ON a.application_id = das.application_id
WHERE dv.deployment_version_id = :deploymentVersionId;
```

Recent in-flight signal-orchestration actions:
```sql
SELECT * FROM workflow_manager_state WHERE active = true ORDER BY created_at DESC LIMIT 200;
```

## Gotcha: no symmetrical "workflow completion" event

`e5-platform-core` publishes `WF_MANAGER_ACTION_WF_FAILURE_NOTIFIED` (a
`WorkflowManagerEvent<AppSignalPayload>`, see
`knowledge/e5-platform-core/_overview.md` NDE/replay-failure mechanism) to a
Kafka topic this service consumes. There is **no** symmetrical
`WF_MANAGER_ACTION_WF_COMPLETION`/`WORKFLOW_COMPLETED` event anywhere —
confirmed absent from `e5-workflow-manager-service`'s
`WorkflowManagerEventType` enum (which has `WORKFLOW_TERMINATED`,
`WORKFLOW_SOFT_TERMINATED`, `WORKFLOW_FAILED`, etc., but no completion
variant) and from `e5-platform-core`. The closest adjacent concept,
`taskCompletionEndPoint` (a Kafka topic name inside the
`workflowManagerTopics` env var, `WorkflowManagerConstants.WORKFLOW_MANAGER_TOPICS_CONFIG`),
is task-level Kafka routing (this service's own inbound topic name), not a
workflow-lifecycle completion notification — don't reach for it expecting
the latter.
