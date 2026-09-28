---
service: e5-deployment-mgmt-service
feature: schedule-enable-disable-version-race
status: active
related_files:
  - domain/usecase/UpdateScheduleStatusUseCase.java
  - domain/usecase/UpdateWorkflowAndScheduleStatusUseCase.java (enableSchedules, disableSchedules)
  - data/repository/ScheduleRepositoryImpl.java (getAllActiveSchedules, getAllSchedulesByDeploymentId)
  - data/repository/DeploymentStateRepositoryImpl.java (findByConfigValue)
  - data/repository/WorkflowRunRepositoryImpl.java (deleteWorkflowRunForDeploymentVersionId)
  - scheduler/task/BaseTask.java (populateWorkflowRuns, createDefaultWorkflowRunEntity)
  - scheduler/task/DailySchedulerTask.java
  - domain/usecase/SaveDeploymentDetailsUseCase.java (cutover transaction)
integrations: []
---

# schedule-enable-disable-version-race

Confirmed root cause of a recurring production incident: a schedule's
trigger gets disabled then re-enabled around the same time as a deployment
version cutover, and the enable silently lands on the wrong (old) version's
schedule — leaving the new version's trigger permanently disabled with no
error surfaced anywhere.

## 2026-09-21 — data model: schedule → workflow_run, not a live re-resolve

`schedule.deployment_version_id` is a live FK — no history. Each new
deployment version gets its **own new `schedule` rows** (new `schedule_id`
per version), correlated across versions only by the shared `trigger_id`
string column (not by FK). The actual "day's list of triggers to fire" is
materialized into `workflow_run`, one row per `(schedule_id,
expected_start_time)` — unique-constrained on that pair — and the FB/trigger
firing mechanism reads only the already-materialized `workflow_run` row; it
never re-resolves `deployment_version_id` at fire time. So whatever
`deployment_version_id` gets snapshotted onto `workflow_run` at generation
time is final for that run.

Three code paths generate `workflow_run` rows, all by copying
`schedule.getDeploymentVersion()` at that instant:
`DailySchedulerTask` (nightly, 23:45 UTC, filters
`getAllActiveSchedules()` = `status=ENABLED AND deploymentVersion.active=true`),
`UpdateWorkflowAndScheduleStatusUseCase.enableSchedules` (immediate,
unconditional — does **not** check `deploymentVersion.active` before
materializing rows), and `GenerateRunRecordsTask` (one-shot, fired right
after a new deployment version is saved).

## 2026-09-21 — root cause: enable-schedule resolves "active version" by side-effecting lookup, not by pinned id

`ScheduleStatusRequest` (the enable/disable gRPC payload) carries only
`kubeNamespace` + `scheduleNames` (trigger ids) + `status` — **no**
`deployment_version_id`. `UpdateScheduleStatusUseCase` resolves the target
version itself, per request, via
`DeploymentStateRepositoryImpl.findByConfigValue("kubeNamespace", ns)` →
`WHERE ds.key='kubeNamespace' AND dv.active=true` — a plain unlocked read,
in its own transaction, fully decoupled from the cutover transaction
(`SaveDeploymentDetailsUseCase.saveDeploymentDetails`, which flips
`active` false→old / true→new atomically in one transaction via
`disableDeploymentVersion` + `updateStatusToDeploymentVersion`). Whichever
version happens to satisfy `active=true` at the exact instant the enable
request's SELECT runs wins — no lock, no `@Version` optimistic-concurrency
column on `deployment_version` or `schedule`, no identifier tying the
enable action to the version the operator actually intended.

Confirmed incident sequence (client/workflow redacted): new version created
with schedule rows `DISABLED` (deploy pipeline sets a `WORKFLOW_SUSPEND`
flag on creation, expecting a later separate enable call); ~54s later an
enable-schedule call landed — but resolved `active=true` against the *old*
version (cutover hadn't committed yet from this read's point of view) and
enabled the old version's schedules instead. Net effect confirmed via DB:
zero `workflow_run` rows ever existed for the new version (its schedule
never left `DISABLED`); the old version's schedules got enabled but were no
longer `active`, so the nightly job's `getAllActiveSchedules()` filter
excluded them too — total ~22h window where no valid
`(ENABLED schedule) × (active version)` pair existed, so **nothing was
materialized/fired for the deployment at all**, despite the nightly job
itself running to completion successfully (confirmed via `scheduler_task`
row, `status=COMPLETED`).

## 2026-09-21 — diagnostic SQL for this failure pattern

Given a `deployment_id`, check for version/schedule/run mismatch:

```sql
-- 1. version history + which one is active now
SELECT deployment_version_id, created_at, active
FROM deployment_version WHERE deployment_id = :deploymentId ORDER BY created_at;

-- 2. today's workflow_run rows per schedule, flag stale snapshot
SELECT sch.schedule_id, sch.status AS schedule_status,
       sch.deployment_version_id AS schedule_current_version,
       wr.expected_start_time, wr.status AS run_status,
       wr.deployment_version_id AS run_snapshotted_version,
       (wr.deployment_version_id = sch.deployment_version_id) AS version_matches
FROM schedule sch
JOIN deployment_version dv ON dv.deployment_version_id = sch.deployment_version_id
LEFT JOIN workflow_run wr ON wr.schedule_id = sch.schedule_id
  AND wr.expected_start_time >= CURRENT_DATE AND wr.expected_start_time < CURRENT_DATE + INTERVAL '1 day'
WHERE dv.deployment_id = :deploymentId
ORDER BY sch.schedule_id, wr.expected_start_time;
```
`workflow_run_id IS NULL` for an `ENABLED` schedule → nothing was ever
generated (this failure mode). `version_matches = false` → a different,
narrower failure mode (stale snapshot survives a
`ConstraintViolationException` that `BaseTask.populateWorkflowRuns` catches
and only logs instead of upserting — also worth checking, but is not what
happened in the confirmed incident above).

## 2026-09-21 — no audit trail exists for schedule enable/disable

`schedule.status` + `schedule.updated_at` only reflect the *current* state
— no history table, no `events` row (the existing `events`/`EventEntity`
table is unrelated: webhook event-type subscriptions, not a domain audit
log). `disableSchedules()` hard-deletes `workflow_run` rows (no soft-delete
flag), so nothing persists in Postgres to reconstruct a disable/enable
timeline after the fact. The only usable trail is application logs:
`disableSchedules()` logs `"Deleted workflow run count: {} for schedule id:
{}"` per disable (present); `enableSchedules()` has **no equivalent
schedule-specific log line at all** (nearby log call
`"shouldPopulateRunRecords : "` is also a logging bug — positional `{}`
arg won't interpolate, and it isn't schedule-specific anyway). Reconstructing
an incident timeline currently requires correlating `deployment_version.created_at`,
`schedule.updated_at`, and log grep — there is no single source of truth.

## 2026-09-21 — fix proposal (not yet implemented)

Recommended, smallest root-cause fix: make `ScheduleStatusRequest` carry an
explicit `deployment_version_id` (the id the caller/pipeline already has
from `createDeploymentVersion`'s response) instead of having the server
guess "whichever version is active now" via `findByConfigValue`.
`UpdateWorkflowAndScheduleStatusUseCase.processRequest` already checks
`deploymentVersionEntity.isActive()` — reject the call if the specific
version isn't active yet (force a caller retry after cutover) rather than
silently resolving against a stale/wrong version. Optionally pair with an
`@Version` optimistic-lock column on `deployment_version` as defense in
depth (turns a silent wrong-row mutation into a loud CAS failure). A more
invasive alternative — folding schedule-enable-state carry-over into the
cutover transaction itself, so a separate out-of-band enable call isn't
needed for the redeploy case — was considered but not recommended as the
first move; only reach for it if the pinned-id fix turns out insufficient
for how the deploy pipeline actually sequences these two calls today.
