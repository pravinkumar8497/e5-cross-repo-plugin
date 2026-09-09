---
integration: e5-platform-core <-> wf-scaffolding
protocol: java-library-api + annotation-processor codegen + shared config schema
direction: wf-scaffolding depends on e5-platform-core
---

# e5-platform-core ↔ wf-scaffolding

`wf-scaffolding` is a downstream app authored against `e5-platform-core` — it's the
practical place to smoke-test any platform-core change end to end (Temporal
worker/engine actually running, real config files, real annotation-processor output).

## What crosses the boundary

1. **Pinned library dependency.** `wf-scaffolding/build.gradle` declares
   `implementation`/`annotationProcessor` on `platform.core:e5-platform-core` at an
   exact Artifactory version — not a live/project dependency. Every fix on the
   `e5-platform-core` side needs: version bump → `artifactoryPublish` (or
   `publishToMavenLocal` for a faster loop) → matching version bump in
   `wf-scaffolding/build.gradle` → rebuild. Skipping this step is the most common way
   to see a "fix didn't work" that's actually just a stale jar.

2. **Annotation-processor codegen contract (UIA).** `E5UserInitiatedActionAnnotationProcessor`
   (in `e5-platform-core`) runs against `wf-scaffolding`'s source at build time. Two
   things must stay in sync between the processor and any app using it:
   - **Skeleton placement:** `<app-root-package>.useraction.<lowercase-bare-action-name>.<ClassName>.java`
     (enforced by the processor's `getClassPath()` — throws if violated).
   - **Generated output package:** `<app-root-package>.useraction.generated`, flat,
     shared across all actions. `E5TemporalClient.resolveUserActionClass` (runtime
     side, in `e5-platform-core`) must derive the exact same package string as
     `E5UserInitiatedActionAnnotationProcessor.getDesiredPackage()` (build-time side) —
     these drifted once (2026-09-09, see `knowledge/e5-platform-core.md`) and caused a
     runtime `ClassNotFoundException` even though codegen had succeeded.

3. **Shared config schema.** `EngineConfig`/`WorkerConfig` fields (e.g.
   `uiActionMappings`, `taskQueueName`, `akstClientConfig`) are validated against
   `*.schema.json` files bundled inside `e5-platform-core`'s jar — `wf-scaffolding`
   supplies YAML that must match both the Java field name *and* the schema's declared
   property name exactly. A field named with a lowercase-then-two-uppercase pattern
   (e.g. the original `uIActionMappings`) silently breaks this because of a
   Lombok/`Introspector.decapitalize` interaction — see `knowledge/e5-platform-core.md`.

## Where to look when something breaks across this boundary

- Runtime `ClassNotFoundException` for a generated UIA action → check the
  processor's `getDesiredPackage()` vs. `E5TemporalClient.resolveUserActionClass`
  agree on the package string, given `wf-scaffolding`'s actual `WorkflowConfig.workflowClass`
  package.
- Config `SchemaFileNotFoundException` or `Unable to find property '...'` → check
  whether `wf-scaffolding` is still resolving a stale `e5-platform-core` jar (version
  bump/publish cycle above) before assuming the config content itself is wrong.
- A skeleton class the annotation processor rejects with "must be present inside its
  own useraction/<actionName> subpackage" → placement convention above, not a
  processor bug.
