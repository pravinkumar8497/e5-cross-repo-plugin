---
service: e5-platform-core
role: library
depends_on: [e5-task-sdk-java, e5-common-utils]
consumed_by: [wf-scaffolding]
features: [user-initiated-action, configuration]
integrations: [e5-platform-core--e5-task-sdk-java, e5-platform-core--wf-scaffolding]
---

# e5-platform-core

Workflow runtime SDK (Temporal-based) — a library consumed by downstream
workflow-author services, not a standalone deployable. Wraps Temporal with
E5-specific primitives: stages, blocks, signals, microworkers, intervention
handling, RPA gateway, event/aggregator pipeline, and (new) User Initiated
Actions (UIA).

## Features

- [user-initiated-action](./user-initiated-action.md) — `@E5UserInitiatedAction`
  codegen, runtime registration/dispatch, akst Task completion routing, workflow
  lifecycle bugs found this session.
- [configuration](./configuration.md) — config field-naming / schema-binding
  gotchas in `ConfigsLoader`.

## Environment notes (repo-wide, not feature-specific)

- Local `./gradlew compileJava` fails with `NoSuchFieldError: JCTree$JCImport ...
  qualid` — a pre-existing Lombok/JDK-21 incompatibility. Can't compile-verify
  changes locally; cross-check method/field signatures via `grep`/`javap`-decompiling
  the relevant jar instead.
- Downstream consumer used for testing this session: `wf-scaffolding` (sibling
  repo) — see `integrations/e5-platform-core--wf-scaffolding.md` for the
  version-pin/publish-cycle relationship.
