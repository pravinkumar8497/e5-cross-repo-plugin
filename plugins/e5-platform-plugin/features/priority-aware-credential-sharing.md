---
feature: priority-aware-credential-sharing
status: active
repos: [e5-deployment-mgmt-service, e5-platform-core, e5-platform-ss-interface, e5-workflow-manager-service]
---

# priority-aware-credential-sharing

One read for the whole thing — everything needed to work on this feature or
fix a bug in it, in any of the four repos it touches, without hunting across
per-repo knowledge files.

## What it is

Multiple workflows can share one pool of login credentials (a "credential
sharing group") instead of each workflow getting its own. Two problems that
creates, and what each repo does about one of them:

1. **Fairness across workflows on the shared queue.** Each workflow
   historically picked its own priority numbers independently — one used
   `1..3`, another `100..300` — so whichever picked smaller numbers always
   won, regardless of actual urgency. **`e5-platform-core`** fixes this:
   each deployment declares its own priority band, normalized onto one
   comparable scale.
2. **Provisioning/capacity for the shared pool.** **`e5-deployment-mgmt-service`**
   owns the group's identity, size, and guaranteed capacity, and rejects a
   deployment that would overcommit it. **`e5-workflow-manager-service`**'s
   devops-pipeline automation writes the group's declared size into tenant
   config. **`e5-platform-ss-interface`** lets an operator resize/rescale a
   whole group through one GraphQL call instead of one-app-at-a-time.

## End-to-end flow

```
devops pipeline config (tenant-common-config.yaml)
   ScaleAppWorkerGitHubConfigHandler.updateTenantSharedCount()      [e5-workflow-manager-service]
   writes credentialSharingGroup.sharedCount (+ multiplier: 1.25 above 20)
        │
        ▼
deployment into the group
   SharedCredentialGroupRepository.upsertMembership()                [e5-deployment-mgmt-service]
   creates/joins shared_credential_group + _member, enforces capacity
        │
        ▼
tasks run on the shared queue
   BaseFunctionBlock.setFunctionBlockOptions()                       [e5-platform-core]
   → PriorityNormalizer.normalize(), if priorityRange is declared
        │
        ▼
operator wants to rescale the whole group
   scaleAppWorker(input: {groupName, groupCount})                    [e5-platform-ss-interface]
   → GetSharedCredentialGroup (DMS) → task → Kafka → (workflow-manager, unimplemented)
```

## e5-deployment-mgmt-service — the group's data model

```
shared_credential_group
  group_id PK, group_name UNIQUE, customer_id FK, application_id FK
  total_count                  -- the credential count, author-controlled
  common_deployment_count      -- guaranteed capacity: MIN(member.deployment_count)
                                --   across members whose deployment_version is active
  deployment_multiplier        -- NUMERIC(6,4); 1.25 = proportional rule, 1 = flat rule, null = not recorded
  ccs_pool_name, ccs_workflow_name  -- identity of the pool; NOT the access token (rotates, isn't identity)
  is_active                    -- operational switch, own RPC, not a deploy side-effect

shared_credential_group_member
  member_id PK, group_id FK, deployment_version_id FK, deployment_count
  UNIQUE(group_id, deployment_version_id)
  -- no liveness flag of its own: live exactly when deployment_version.active is
```

**Capacity rule is MIN, not MAX, across members** — the smallest
`deployment_count` among active members, recomputed on every
`upsertMembership` (denormalized, read on every deployment, never drift). A
deployment whose `total_count` would exceed this is rejected: those
credentials would have nowhere to run on the least-provisioned workflow.
(An initial high-water-mark/MAX design was corrected mid-build — the real
requirement is MIN, not `min(existing, incoming)` either, which would
ratchet down permanently.)

**Check order** in both `upsertMembership` (write) and `validateMembership`
(read-only pre-check, same rules, used at `CreateDeploymentVersion` time to
fail before pods exist): `requireCompletePoolIdentity` → `requireSameTenant`
→ `requireSameApplication` → `requireActiveGroup` → `requireSamePool` →
`requireWithinCapacity`. Tenant ownership checked first on purpose — a
cross-tenant name collision is worse than a count mismatch even when counts
happen to agree.

**`is_active`** is its own RPC (`SetSharedCredentialGroupActive`), not a
deploy side-effect — a deployment into a deactivated group is rejected and
never silently reactivates it. Returns `active_member_count` so a
deactivation call surfaces how many live workflows it affects.

**`GetSharedCredentialGroup`** (read-only, added for the interface's group
scaling, below):
```protobuf
rpc GetSharedCredentialGroup(GetSharedCredentialGroupRequest)
    returns (GetSharedCredentialGroupResponse);
message GetSharedCredentialGroupResponse {
  bool found = 1; bool active = 2;
  int32 total_count = 3; int32 common_deployment_count = 4;  // returned, not yet checked by any caller
  string message = 5;
}
```
No existing RPC was keyed on `group_name` alone (`validateScaleAppWorkerVersionRequirement`,
`getFbWorkerPodDetails`, `getAllActiveWorkflowManagerRunResponse` are all
`(deploymentVersionId, appName)`) — this one is, reusing the existing
`findGroupByName` query, no new table.

**Migration history:** this whole schema was consolidated from seven
in-progress migration files into one,
`V1_2026_10_12_001__create_shared_credential_group.sql` — none had reached
a base branch, so there was nothing worth preserving separately. One
pre-consolidation file's backfill read `member.active`/`member.enabled`
columns already deleted by later review fixes in the same chain — would
have aborted Flyway on any genuinely clean DB. Invisible before
consolidation because nothing had actually run the migrations yet (no
TestContainers in this repo).

**Local dev gotcha, unrelated to this feature's code but blocks testing it
locally:** no migration in this repo creates ~24 foundational tables
(`customer`, `application`, `deployment_version`, etc.) — they predate
Flyway tracking and the baseline was never captured. `baselineOnMigrate(true)`
in `Main.java` doesn't help on a genuinely empty DB (only helps against an
already-populated one). There is no way to bootstrap a working local DB from
empty without an existing schema dump — get one from someone already
running this service rather than reconstructing ~24 tables from Hibernate
entity mappings (started down that path once, abandoned once a real DB
became available — real risk of subtle drift from the actual schema).

## e5-platform-core — priority normalization

`WorkerConfig.priorityRange` (optional; `null` = untouched, exactly as
before this feature) declares `minimumPriority..maximumPriority`.
`PriorityNormalizer.normalize()` stretches it onto `1..Long.MAX_VALUE` via
`BigInteger` — a span of ~99,000 already overflows plain `long`
multiplication, not avoidable with primitive math. `min >= 1` is
**load-bearing for overflow safety**, not just policy: a negative `min`
overflows `max - min` toward `2^63`.

Single chokepoint for all of it: `BaseFunctionBlock.setFunctionBlockOptions`
— every function block's priority resolves there before delegation,
`priorityRange` declared or not.

**The regression this surfaced (PR element5inc/e5-platform-core#635):** the
platform-wide convention — stated in `PriorityNormalizer`'s own javadoc —
is that an unprioritized function block defaults to `Long.MAX_VALUE` (least
urgent, runs last). Wiring normalization needed a separate `prioritySpecified`
boolean (still correct and necessary — it keeps an unprioritized task out of
`normalize()` entirely, since `Long.MAX_VALUE` would otherwise blow past
almost any declared band and throw a spurious failure), but the literal
picked for "not specified" was wrong — `1L` (most urgent) — in three spots:
the unconditional fallback in `BaseFunctionBlock` (predates this feature;
confirmed via the PR diff that line was untouched context, not new), the
`priorityRange != null` branch in the same method (new), and
`RPADelegator.resolvePriority` (new, picked `1L` specifically "to agree
with" the first, wrong one). Harmless in one workflow's own queue (every
unprioritized task got the same wrong constant, so relative order didn't
change) until credential sharing puts two workflows on the same queue —
then a non-subscribed workflow's unprioritized tasks outrank a subscribed
workflow's properly-ranged explicit priorities. All three fixed to
`Long.MAX_VALUE`. A pre-existing test asserting the wrong value (`1L`,
despite its own name saying "lowest priority") was rewritten.

**Side effect found while placing the failure correctly relative to
events:** `FUNCTIONBLOCK_STARTED` now fires *before* `setFunctionBlockOptions`
runs (both branches in `BaseFunctionBlock.execute()`), so a block always
gets a STARTED event even when a priority-range failure happens — previously
the `finally` block's unconditional `FUNCTIONBLOCK_ENDED` could fire with no
matching STARTED at all. Verified real (not cosmetic) by tracing
`EventEmitter.send` → `Workflow.sideEffect` → `KafkaProducer.send()`, which
serializes synchronously before buffering. Consequence: the STARTED event's
`functionBlockInput.options` now shows pre-resolution values, not the
defaulted/normalized ones the block actually runs with.

**Schema gotcha:** `WorkerConfig.schema.json` has `additionalProperties: false`
and is enforced at worker startup, not just design-time — `priorityRange`
had to be added there too. It's `type: ["object", "null"]` with its own
nested `required: [minimumPriority, maximumPriority]`, so omitting the key
passes but a half-declared range fails schema validation before
`PriorityNormalizer.validatePriorityRange` ever runs.

## e5-platform-ss-interface — scaling a whole group through scaleAppWorker

`ScaleAppWorkerRequest` accepts either the original four per-app fields
(`deploymentVersionId`/`appName`/`minReplicas`/`maxReplicas`) or
`groupName`/`groupCount` — not both, not neither. All six fields are
nullable (GraphQL `Int`/`String`, Java `Integer` not `int`) specifically so
group mode can omit the per-app fields entirely rather than send meaningless
`0`s; `isModeValid()`'s "was this given" checks depend on `null` being
distinguishable from a legitimate `0` (there's a test asserting
`minReplicas: 0` is valid in app mode, so `0` can never mean "absent").

```
scaleAppWorker(input: {groupName, groupCount})
  → request.isModeValid() check
  → isGroupMode() → scaleCredentialGroup(request)
      → getSharedCredentialGroup(groupName)        [DMS RPC, above]
          found=false → FAILED "No credential group named '<name>'..."
          active=false → FAILED "...is deactivated and cannot be scaled."
      → same task mechanism as per-app mode, fbPayload = {groupName, groupCount}
```
None of the per-app checks run for a group request — they're all keyed on
`(deploymentVersionId, appName)`.

**Deliberately not decided here, real gap for whoever implements the
workflow-manager side:** no capacity check against `common_deployment_count`
(returned by DMS, unused); whether `groupCount` changes the group's actual
credential count or just running replicas; one task for the whole group vs.
fan-out per member (currently: one task for the whole group). See "Cross-repo
contracts" below.

**The validation bug this surfaced, found via manual testing, not any
automated test:** the "exactly one target" rule was originally only a
`@AssertTrue` on `ScaleAppWorkerRequest`, enforced via `@Valid @Argument` +
`@Validated` on the controller. **This does not fire for a live GraphQL
request** — Spring for GraphQL invokes `@MutationMapping` handlers through
its own reflection-based dispatch, not the Spring MVC/AOP pipeline
`@Validated` method-level validation relies on. A request setting
`groupName` + `groupCount` + `appName` together was accepted and queued in
production use, despite a passing unit test for the same rule — that test
called `Validator.validate(request)` directly, bypassing the controller
entirely, proving the rule was *expressible*, not that it was *enforced*.
Fixed by moving the check into `ScaleAppWorkerServiceImpl.scaleAppWorker` as
a plain `if (!request.isModeValid())` — same style every other check in
that class already uses.

**A second bug the `Integer` migration itself introduced:** widening
`minReplicas`/`maxReplicas` from `int` to `Integer` silently broke three
pre-existing `==`/`!=` comparisons that had relied on one side being
primitive to force auto-unboxing — with both sides boxed, they became
reference comparisons, wrong for any value outside `-128..127`. Fixed with
`Objects.equals(...)`. Worth re-checking this exact pattern any time a
numeric field widens from primitive to boxed for an unrelated reason.

## e5-workflow-manager-service — devops pipeline config, and an unfixed bug

`ScaleAppWorkerGitHubConfigHandler.updateCredentialSharingGroupSharedCount`
writes `credentialSharingGroup.sharedCount` into a tenant's `fb_worker`
config entry as part of the pipeline automation that commits updated
tenant config back to GitHub. Its own javadoc also promises: *"Sets
multiplier to 1.25 when the count is above 20, and removes it otherwise"* —
but the method body never touches `multiplier` at all. Two tests
(`..._setsMultiplierAboveTwenty`, `..._removesMultiplierAtOrBelowTwenty`)
correctly encode the documented behavior and fail against the real
implementation — surfaced as a CI image-build failure, not a flaky test.
**Not yet fixed.** The fix, at the exact spot `sharedCount` is set:
```java
credentialSharingGroup.put("sharedCount", sharedCount);
if (sharedCount > 20) {
    credentialSharingGroup.put("multiplier", 1.25d);
} else {
    credentialSharingGroup.remove("multiplier");
}
```
Note the YAML key here is `multiplier`, not `deploymentMultiplier` — a
different layer of the same concept from DMS's
`shared_credential_group.deployment_multiplier` column; don't conflate them.

This repo also owns the actual group-scaling behavior that the interface's
`scaleAppWorker` group mode hands off to — see "Cross-repo contracts" below
for the state of that handoff.

## Cross-repo contracts

Full wire-level detail lives in `integrations/`, not repeated here:

- `integrations/e5-deployment-mgmt-service--e5-platform-ss-interface.md` —
  the gRPC contract (which RPCs the interface actually calls, and that the
  interface keeps its own un-synced copy of the proto file — `SetSharedCredentialGroupActive`
  has no mirror there as of this writing).
- `integrations/e5-platform-ss-interface--e5-workflow-manager-service.md` —
  the Kafka/task payload contract, explicitly flagged **half-implemented**:
  the group-mode shape the interface sends (`{groupName, groupCount}`) has
  no confirmed consumer on the workflow-manager side yet.

## Open items (as of 2026-10-27)

- Workflow-manager's actual group-scaling logic: not started (scoped to
  another team member; the interface task above stopped at emitting the
  task).
- Interface-side capacity check (`groupCount` vs. `common_deployment_count`):
  not implemented — deliberately deferred, data is available if needed.
- `ScaleAppWorkerGitHubConfigHandler`'s multiplier bug: not fixed.
- DMS local-dev baseline-schema gap: not fixable from within this repo; needs
  an external schema dump.
