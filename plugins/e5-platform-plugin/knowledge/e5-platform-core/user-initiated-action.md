---
service: e5-platform-core
feature: user-initiated-action
status: active
related_files:
  - src/main/java/com/e5/platform/core/annotation/processor/E5UserInitiatedActionAnnotationProcessor.java
  - src/main/java/com/e5/platform/core/workflow/BaseUserInitiatedActionWorkflow.java
  - src/main/java/com/e5/platform/core/wre/util/E5TemporalClient.java
  - src/main/java/com/e5/platform/core/wre/engine/worker/TemporalWorker.java
  - src/main/java/com/e5/platform/core/utility/TaskTransportUtil.java
integrations: [e5-platform-core--e5-task-sdk-java, e5-platform-core--wf-scaffolding]
---

# User Initiated Action (UIA)

## 2026-09-09 — end-to-end shape

`@E5UserInitiatedAction`-annotated skeleton classes get build-time-processed by
`E5UserInitiatedActionAnnotationProcessor` into a concrete Temporal workflow
(extends `com.e5.platform.core.workflow.BaseUserInitiatedActionWorkflow`) plus a
per-action `@WorkflowInterface`. Runtime: `TemporalWorker` resolves and registers
each configured action by bare name (`EngineConfig.uiActionMappings`), on a
dedicated task queue (`EngineConfig.taskQueueName`); `E5TemporalClient.launchUserAction(Task)`
starts one by name via an untyped stub.

**Skeleton placement convention:** must live at
`<app-root-package>.useraction.<lowercase-bare-action-name>.<ClassName>.java`, e.g.
`e5.sampleworkflow.useraction.retrigger.RetriggerAction.java` — enforced by
`E5UserInitiatedActionAnnotationProcessor.getClassPath()`. Generated output (the action
class + its `*Workflow` interface) lands in a flat, shared
`<app-root-package>.useraction.generated` package regardless of the skeleton's
subpackage.

**Do not give `BaseUserInitiatedActionWorkflow` a shared `@WorkflowInterface`.** An
earlier version implemented `IBasicTemporalWorkflow` (one shared, unnamed workflow
type) — every generated subclass inherited it in addition to its own per-action
interface, so registering more than one action on the same Temporal worker threw
`TypeAlreadyRegisteredException`. Fixed by making `startAction(AkstClientTask)` a plain
concrete method the generated subclass's own interface is satisfied by via inheritance
(ordinary Java interface-satisfaction, no override needed in the subclass).

## 2026-09-09 — `resolveUserActionClass` package-derivation bug (fixed)

`E5TemporalClient.resolveUserActionClass` used to strip the last segment off
`workflowClass.getPackageName()` before appending `.useraction.generated`, assuming
the workflow class always lives one package level below the app root. Doesn't hold
when the workflow class *is* the app root package (e.g. `e5.sampleworkflow.SampleWorkflow`
→ wrongly resolved to `e5.useraction.generated` instead of
`e5.sampleworkflow.useraction.generated`). Fixed to append directly to
`workflowClass`'s own package, matching
`E5UserInitiatedActionAnnotationProcessor.getDesiredPackage()`'s build-time convention
exactly (`"e5." + appName + ".useraction.generated"`).

## 2026-09-09 — akst Task→TaskComplete routing needs `routeCompletion`, not `routeTask`

`TaskTransportUtil.buildTask` never sets `TaskTransport.metaInfo` (a completion
response carries no routing metadata of its own), but the SDK's
`TaskProducerService.routeTask(TaskTransport, endpointId)` unconditionally calls
`taskTransport.getMetaInfo().setRoutingPath(...)` → NPE. Also, `TaskTransport.schema.json`
requires `metaInfo.userMetaInfo`, so even stubbing an empty `MetaInfo` only trades the
NPE for a schema-validation failure.

Fix implemented across both repos: added `routeCompletion(TaskTransport, endpointId)` to
`e5-task-sdk-java`'s `ITaskProducerService`/`TaskProducerService`/`MockProducerService`/
`TaskProcessor` — resolves the destination `Endpoint` by id directly (no
`producers:`-list/`getRoutingPaths` lookup, no `TaskTransport.schema.json` validation),
publishes straight to Kafka keyed by `taskInfo.getTaskId()` instead of
`metaInfo.userMetaInfo.taskIdentifier`. `platform-core`'s `TaskTransportUtil.routeCompletion`
delegates to it. `sendUserInitiatedActionResponse` routes through the existing
`"akstQueue"` endpoint, whose topic `ConfigsLoader.updateAkstEndpoint(WorkerConfig)`
already keeps pointed at `WorkerConfig.akstEndpoint` on every worker startup (same
endpoint `UpdateTask`/`CancelTask` already use) — no new endpoint config needed.
Both repos need version-bump + publish for downstream consumers to see this — see
`integrations/e5-platform-core--e5-task-sdk-java.md`.

## 2026-09-09 — `BaseUserInitiatedActionWorkflow.startAction`: two bugs found by review

1. **Completion event always reported `WORKFLOW_SUCCESS`.** The `finally` block that
   sends `UIA_ENDED` hardcoded `WorkflowEndStatus.WORKFLOW_SUCCESS` regardless of
   whether pre-validation, execution, or post-validation actually failed — any failed
   UIA silently reported success downstream. Fixed by tracking a `uiaStatus` var set in
   every branch (pre-validation error / success / post-validation error / exception)
   and mapping it to `WORKFLOW_SUCCESS` only when it's `E5_UIA_SUCCESS`, else
   `WORKFLOW_FAILED`.

2. **`asyncExecutor.closeAllHandlers()` placed too late.** Putting it in the outer
   `finally` (right before `sendCompletionEvent`) does correctly order it before the
   `UIA_ENDED` *send* — but `postValidate(output)` and `buildOutputTaskPayload(output,
   ...)` run earlier in the `try`, so they can still serialize an `output` whose
   async-block-populated fields haven't resolved yet. `BaseWorkflow.startWorkflow`'s
   working pattern calls `closeAllHandlers()` immediately after `executeWorkflow()`
   returns, before touching the result at all — moved the UIA call to the same spot,
   right after `execute()` and before `postValidate()`. (`closeAllHandlers()` →
   `E5PromiseRepository.resolveAllPromises()`, which does block synchronously via
   `Async.function(...).get()` in a loop — it's real synchronization, not a no-op.)

**Open question, not yet resolved:** if the actual symptom is "`UIA_ENDED` arrives on
Kafka before an async block's *own* completion event", that's a Kafka
publish-ordering/flush question `closeAllHandlers()` can't address at all — it only
guarantees Temporal-side promise completion, not that events published from inside a
resolved promise have already been acked to the broker before a later publish on the
main thread.
