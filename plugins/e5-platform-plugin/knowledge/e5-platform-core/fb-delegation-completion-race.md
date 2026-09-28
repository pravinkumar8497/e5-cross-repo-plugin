---
service: e5-platform-core
feature: fb-delegation-completion-race
status: active
related_files:
  - src/main/java/com/e5/platform/core/executionblock/functionblock/delegator/RPADelegator.java
  - src/main/java/com/e5/platform/core/token/DelegationCompletionReconciler.java
  - src/main/java/com/e5/platform/core/token/DelegationQueueReconciler.java
  - src/main/java/com/e5/platform/core/token/DelegationTokenUpdates.java
  - src/main/java/com/e5/platform/core/converter/E5Token.java
  - src/main/java/com/e5/platform/core/wre/engine/worker/FunctionBlockInboundActionHandler.java
  - src/main/java/com/e5/platform/core/wre/util/E5TemporalClient.java (completeActivity)
integrations: []
---

# fb-delegation-completion-race

Investigated while root-causing a production incident: a workflow stuck
waiting for an FB response even though the FB worker completed the task in
~2s. There is one unified DB table (`e5_token`) tracking FB delegation
across both Temporal activities involved (dispatch + execution), and a
known, already-fixed race between FB completion arriving and the execution
activity registering — plus one residual gap found while verifying the fix
actually covers this incident.

## 2026-09-21 — schema: `e5_token` is the single source of truth, not separate queue/execution tables

```
e5_token
  tokenId            UUID PK
  temporalToken       varchar(500) UNIQUE  -- Activity-1's token at dispatch, overwritten with Activity-2's token on registration
  workflowId, workflowRunId
  activityStatus      ACTIVE | PICKED | PAUSED | COMPLETED | FAILED | TIMEOUT
  delegationId         (unique)             -- FB task correlation id, the actual lookup key for completion callbacks
  completionReceived  boolean default false -- true if TaskComplete arrived before Activity-2 registered
  completionPayload   TEXT                  -- buffered FB payload when completionReceived=true
  pickReceived        boolean default false
```
Row created synchronously in `RPADelegator.delegate()`'s create-flow branch
**before** the task is routed to the FB worker (`tokenService.insertToken`
then `taskProcessor.routeTask`) — not lazily on pickup. Updated (not a new
row) to Activity-2's token + `PICKED` status in
`RPADelegator.getDelegationResponse` → `DelegationCompletionReconciler.registerExecutionActivity`,
which happens later/asynchronously relative to dispatch, when the Temporal
workflow engine schedules Activity-2.

## 2026-09-21 — the dispatch-order race was already found and fixed (commit 5de22d35)

`DelegationCompletionReconciler`'s own class Javadoc: *"Reconciles Activity-2
completion events with execution-activity token registration, using
row-level locking to eliminate the race between TaskComplete and Activity-2
creation."* Commit `5de22d35` ("fix: race condition occuring during fb
delegation and pause and resume activity #575", 2026-06-24) is present on
every release branch checked (`release/v1.8.13` through `release/v1.9.6`).
Concurrency-tested directly:
`DelegationCompletionReconcilerTest.concurrentCompletionAndRegistration_doesNotLosePayload`
runs `reconcileTaskComplete` and `registerExecutionActivity` concurrently via
an `ExecutorService` and asserts the payload is never lost.

**Completion handling is one method, one lookup, one lock — no
token-vs-execution-activity fallback chain.** `FunctionBlockInboundActionHandler.completeFunctionBlockActivity`
→ `DelegationCompletionReconciler.reconcileTaskComplete(delegationId, payload)`
looks up `e5_token` by `delegationId` under row lock and branches on
`activityStatus`:

| `activityStatus` at arrival | outcome |
|---|---|
| `COMPLETED` | `IGNORE_DUPLICATE`, no-op |
| `TIMEOUT`/`FAILED` | `REJECT_INVALID_STATE`, dropped |
| `ACTIVE`/`PAUSED` (Activity-2 not yet registered) | `BUFFER_UNTIL_EXECUTION_REGISTERED` — sets `completionReceived=true` + `completionPayload`, does **not** touch Temporal yet |
| `PICKED` (Activity-2 already registered) | `COMPLETE_EXECUTION_ACTIVITY` — completes Temporal immediately using the current `temporalToken` |
| no row found for `delegationId` | `REJECT_INVALID_STATE`, `LoggerUtil.logError("No token found for delegationId during TaskComplete reconciliation...")` — silently dropped, **no retry, no requeue** |

## 2026-09-21 — residual gap: PICKED-path Temporal completion failure has no buffer/retry

Unlike the `ACTIVE`/`PAUSED` path (which buffers to DB before touching
Temporal), the `PICKED` path calls
`e5TemporalClient.completeActivity(decodedTaskToken, response)` directly with
no DB buffer first. `E5TemporalClient.completeActivity`
(`wre/util/E5TemporalClient.java:123-133`) swallows
`ActivityNotExistsException`/`ActivityCompletionFailureException` and
returns `false`. The caller
(`FunctionBlockInboundActionHandler.java:148-166`) on `false` just logs
`"Temporal completion failed for delegationId: {}; token state unchanged
for retry"` — **the comment says "for retry" but nothing actually
retries or re-buffers.** The FB's completion payload is lost entirely; the
token stays `PICKED` forever; the workflow hangs exactly like the "FB
completed but workflow never unblocks" incident symptom, even with the
`5de22d35` fix in place. This is the branch to check first when this
symptom recurs on a runtime that's confirmed to already have the fix.

## 2026-09-21 — diagnostic SQL

```sql
SELECT "tokenId", "delegationId", "temporalToken", "workflowId", "workflowRunId",
       "activityStatus", "completionReceived", "completionPayload", "pickReceived"
FROM e5_token WHERE "workflowId" = :temporalWorkflowId ORDER BY "tokenId";
```
- `activityStatus=ACTIVE/PAUSED`, `completionReceived=true`, payload populated
  → fix worked (completion correctly buffered), but **Activity-2 never
  registered** — a different bug, upstream of this reconciler (workflow
  itself stalled before scheduling Activity-2).
- `activityStatus=PICKED`, `completionReceived=false`, payload null →
  Activity-2 registered, completion callback hit `COMPLETE_EXECUTION_ACTIVITY`
  — check worker logs for `"Temporal completion failed for delegationId"`
  (the residual gap above) or `"No token found for delegationId during
  TaskComplete reconciliation"`.
- No `@Version`/history on `E5Token` (`@EntityListeners(E5StateAuditListener)`
  may carry audit timestamps in a separate table — check that if row-level
  timing is needed and `tokenId` ordering isn't enough).

**Also confirm the deployed `e5-platform-core` artifact version before
assuming this incident is the residual gap** — `5de22d35` might not be in
the version actually running; check the resolved dependency version in the
consuming service's build first.
