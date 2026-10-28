---
service: e5-deployment-mgmt-service
role: service
depends_on: []
consumed_by: [e5-platform-ss-credential-monitor, e5-workflow-manager-service, e5-platform-ss-interface]
features: [credential-release-appname, schedule-enable-disable-version-race, platform-version-lookup]
integrations: [e5-deployment-mgmt-service--e5-workflow-manager-service, e5-deployment-mgmt-service--e5-platform-ss-interface]
platform_features: [priority-aware-credential-sharing]
---

# e5-deployment-mgmt-service

gRPC server (Java, no Spring, Hibernate/Postgres/Flyway, Quartz scheduler). Owns
deployment/customer/subscription/scheduling data.

## Features

- [credential-release-appname](./credential-release-appname.md) — resolving
  `appName` on credential-assignment records instead of deriving it ambiguously
  from `resourceName`/pod name.
- [schedule-enable-disable-version-race](./schedule-enable-disable-version-race.md) —
  confirmed root cause of a recurring incident: enable-schedule resolves the
  "active" deployment version by an unlocked lookup that races deployment
  cutover, silently enabling the wrong version's trigger.
- [platform-version-lookup](./platform-version-lookup.md) — how to resolve
  `e5-platform-core`/FB-core version per `deployment_version_id` via the
  generic `dependency` table; used repeatedly in incident triage.
## Platform features (cross-repo — see `features/`, not duplicated here)

- [priority-aware-credential-sharing](../../features/priority-aware-credential-sharing.md) —
  this repo's piece: `shared_credential_group`/`_member` tables, capacity
  rule, CCS pool identity, `is_active`, and the `GetSharedCredentialGroup`
  read RPC added for `e5-platform-ss-interface`. Also spans
  `e5-platform-core`, `e5-platform-ss-interface`, `e5-workflow-manager-service`.

## Environment notes (repo-wide, not feature-specific)

- **No baseline-schema migration exists for ~24 foundational tables**
  (`customer`, `application`, `deployment`, `deployment_version`,
  `subscription`, `schedule`, `workflow_run`, and others) — confirmed by
  searching all ~55 migration files in `src/main/resources/db/migration` for
  a `CREATE TABLE` of any of them; none exists. The earliest migration
  (`V1_2025_07_09_001__initial_dummy_select.sql`, in a folder literally named
  `intialcommit`) is a one-line `SELECT 1` placeholder. This means these
  tables existed in every real environment *before* Flyway tracking started
  on this repo, and that pre-Flyway baseline was never captured anywhere in
  version control. `Main.flyWayMigration()`'s `.baselineOnMigrate(true)`
  does **not** help here — it only activates against an already-populated
  schema; on a genuinely empty local Postgres, Flyway just tries to apply
  everything from version zero and fails immediately on the first migration
  that references one of these tables (`V1_2025_07_09_002`, FK to
  `deployment_version`). **There is no way to bootstrap a fully-working local
  DB from empty without an existing schema dump from someone already running
  it** — ask around for one rather than reconstructing the baseline from
  Hibernate entity mappings (started down that path once; it's a ~24-table,
  multi-hour reconstruction with real risk of subtle type/constraint drift
  from the real schema, abandoned once a real dump/DB became available).
- **Flyway checksum mismatches on old migration files happen in practice** —
  hit one on `V1_2026_06_03_001__schedule_expression_and_dates.sql`
  (committed by a teammate, later edited by someone else after it had
  already run against a shared local DB). Symptom: `FlywayValidateException:
  Migration checksum mismatch`. Fix is `flyway.repair()` — only rewrites the
  checksum bookkeeping in `flyway_schema_history`, touches no actual table
  data — but `Main.java` only ever calls `.migrate()`, never `.repair()`.
  There's no Flyway CLI available by default either. Workaround: a throwaway
  one-off class reusing `Main`'s exact Flyway config but calling `.repair()`
  instead, run once via the project's own resolved Gradle runtime classpath,
  then deleted.

- **Branch convention:** this repo uses `release/vX.Y.Z` branches, not `main`, as
  the actual current line of development — `main` and even mid-range release
  branches can be stale by dozens of commits. Always check
  `git branch -a | grep release` and diff against the highest version number
  before branching for new work.
- **Local build:** `./gradlew test` works fine in this environment (no external
  dependency auth needed for this repo specifically) — unlike its downstream
  consumers, which need `e5-task-sdk-java` from a private Artifactory.
