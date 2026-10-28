---
service: e5-platform-core
role: library
depends_on: [e5-task-sdk-java, e5-common-utils]
consumed_by: [wf-scaffolding]
features: [user-initiated-action, configuration, fb-delegation-completion-race]
integrations: [e5-platform-core--e5-task-sdk-java, e5-platform-core--wf-scaffolding]
platform_features: [priority-aware-credential-sharing]
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
- [fb-delegation-completion-race](./fb-delegation-completion-race.md) — the
  `e5_token`/`DelegationCompletionReconciler` dispatch-vs-completion race
  (fixed in `5de22d35`), plus a residual unbuffered-completion gap found
  while verifying that fix against a stuck-workflow incident.
## Platform features (cross-repo — see `features/`, not duplicated here)

- [priority-aware-credential-sharing](../../features/priority-aware-credential-sharing.md) —
  this repo's piece: `priorityRange` on `WorkerConfig`, normalized onto
  `1..Long.MAX_VALUE` so workflows sharing a credential pool compete on one
  comparable scale. Also spans `e5-deployment-mgmt-service`,
  `e5-platform-ss-interface`, `e5-workflow-manager-service`.

## Environment notes (repo-wide, not feature-specific)

- Local `./gradlew compileJava` fails with `NoSuchFieldError: JCTree$JCImport ...
  qualid` on JDK 21 — this is specifically Lombok `1.18.26` (the version this
  repo pins) being incompatible with JDK 21's annotation processing, **not**
  an unfixable environment gap. Workaround that actually compiles and runs
  tests: point `JAVA_HOME` at a JDK **17** install (e.g. Corretto 17) instead
  of 21 before invoking `./gradlew` — confirmed working for `compileJava`,
  `compileTestJava`, and running specific test classes. `e5-deployment-mgmt-service`
  and `e5-platform-ss-interface`, by contrast, pin Lombok `1.18.34` and
  genuinely need JDK 21 (their own Gradle toolchain declares it) — don't
  cross-apply this JDK-17 workaround there.
- Downstream consumer used for testing this session: `wf-scaffolding` (sibling
  repo) — see `integrations/e5-platform-core--wf-scaffolding.md` for the
  version-pin/publish-cycle relationship.
