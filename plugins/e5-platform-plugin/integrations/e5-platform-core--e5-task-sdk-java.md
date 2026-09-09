---
integration: e5-platform-core <-> e5-task-sdk-java
protocol: java-library-api
direction: e5-platform-core depends on e5-task-sdk-java
---

# e5-platform-core ↔ e5-task-sdk-java

## What crosses the boundary

`e5-platform-core` calls into `e5-task-sdk-java`'s `TaskProcessor` /
`ITaskProducerService` to route `TaskTransport` messages (akst tasks) to Kafka.
`TaskTransportUtil` (in `e5-platform-core`) is the sole call site.

- **Request-side routing** (`routeTask`) — needs a fully-populated
  `TaskTransport.metaInfo.userMetaInfo`; resolves source/destination endpoints via
  `TaskProducerConfig`'s `producers:` list, matched by `taskInfo.taskType`.
- **Completion-side routing** (`routeCompletion`, added 2026-09-09) — for
  `TaskComplete` responses that carry no routing metadata of their own. Resolves the
  destination `Endpoint` directly by `endpointId` (no `producers:`-list lookup, no
  `TaskTransport.schema.json` validation — that schema requires `metaInfo`, which a
  completion transport doesn't have), publishes to Kafka keyed by
  `taskInfo.getTaskId()` instead of `metaInfo.userMetaInfo.taskIdentifier`.

## Contract to keep in sync

`routeCompletion` exists on both sides of the boundary and must match:
- `e5-task-sdk-java`: `ITaskProducerService.routeCompletion` /
  `TaskProducerService.routeCompletion` / `MockProducerService.routeCompletion` /
  `TaskProcessor.routeCompletion`.
- `e5-platform-core`: `TaskTransportUtil.routeCompletion` (thin wrapper, delegates
  straight through).

If either side's `routeCompletion` signature or semantics change, the other needs a
matching update — there's no schema/contract test catching drift between them
currently.

## Version-pin gotcha

`e5-platform-core` doesn't build against `e5-task-sdk-java` as a live/project
dependency — it's a pinned Artifactory version. A fix in `e5-task-sdk-java` is
invisible to `e5-platform-core` (and anything downstream of it) until: bump
`e5-task-sdk-java`'s version, publish, bump the dependency version in
`e5-platform-core`'s own `build.gradle`, then repeat that whole publish cycle one
level up for `e5-platform-core` itself. See
`e5-platform-core--wf-scaffolding.md` for the next hop in this chain.
