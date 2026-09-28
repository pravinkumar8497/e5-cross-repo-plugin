---
service: e5-deployment-mgmt-service
feature: platform-version-lookup
status: active
related_files:
  - data/repository/DependencyRepositoryImpl.java (findPlatformCoreVersionByWorkflowVariantVersion, findAppWorkerFbCoreVersionByDeploymentInfo, findAppWorkerFbCoreVersionWorkflowVariantPlatformCoreVersionByDeploymentInfo)
  - data/entity/DependencyEntity.java
integrations: []
---

# platform-version-lookup

How to resolve, per `deployment_version_id`, which `e5-platform-core` (and
`e5_rf_fb_worker_core` / `nonrpa.worker.core`) version a workflow was built
against. Came up repeatedly during incident triage (matching a stuck-workflow
symptom against a known platform-core bugfix version).

## 2026-09-21 — versions live in the generic `dependency` table, not on deployment_version

Not a column on `deployment_version`/`deployment_config`/`workflow_variant`/
`workflow_sku`. Stored as key/value rows in `dependency`
(`id, service_id, service_type, key, value` — `value` is plain `text`
despite the entity's `@Type(type="jsonb")` annotation; DDL declares `text`).

- **Platform-core version**: scoped to the *workflow variant* —
  `dependency.service_type = 'e5-workflow-variant'`,
  `dependency.key = 'e5-platform-core'`, `service_id = workflow_variant.variant_id`.
  Path: `deployment_version.workflow_variant_id → workflow_variant.variant_id → dependency.service_id`.
- **FB core version**: scoped to the *app worker* (an app can have multiple
  worker apps under one deployment version) —
  `dependency.service_type = 'e5-app-worker'`,
  `dependency.key = 'e5_rf_fb_worker_core'`, `service_id = application_worker.app_worker_id`.
  Path: `deployment_version.deployment_version_id → deployment_version_application_worker → application_worker.app_worker_id → dependency.service_id`.
  (`nonrpa.worker.core` is a third key on the same `e5-app-worker` scope.)

This is the "technology check in deployment version" (#245) plumbing —
`application_worker.technology` + these `dependency` rows are used together in
`ValidateScaleAppWorkerUsecase`/`UpdateAppStatusUseCase`/`ValidateAppStatusUseCase`
to gate behavior by platform-core version via `ComparableVersion` comparisons.

## 2026-09-21 — ready SQL

Platform-core version + client/workflow name, given a list of `deployment_version_id`:

```sql
SELECT dv.deployment_version_id, c.name AS client_name, ws.name AS workflow_name,
       d.value AS platform_core_version
FROM deployment_version dv
JOIN customer c ON c.customer_id = dv.customer_id
JOIN workflow_sku ws ON ws.workflow_sku_id = dv.workflow_sku_id
JOIN workflow_variant wv ON wv.variant_id = dv.workflow_variant_id
LEFT JOIN dependency d ON d.service_id = wv.variant_id
  AND d.service_type = 'e5-workflow-variant' AND d.key = 'e5-platform-core'
WHERE dv.deployment_version_id IN (:list);
```

FB core version (per app, given one `deployment_version_id`):

```sql
SELECT dv.deployment_version_id, a.name AS app_name, aw.version AS app_worker_version,
       d.value AS fb_core_version
FROM deployment_version dv
LEFT JOIN deployment_version_application_worker dvaw ON dv.deployment_version_id = dvaw.deployment_version_id
LEFT JOIN application_worker aw ON dvaw.app_worker_id = aw.app_worker_id
LEFT JOIN application a ON aw.application_id = a.application_id
LEFT JOIN dependency d ON d.service_id = aw.app_worker_id
  AND d.key = 'e5_rf_fb_worker_core' AND d.service_type = 'e5-app-worker'
WHERE dv.deployment_version_id = :deploymentVersionId;
```

`customer_id`/`workflow_sku_id` are denormalized directly onto
`deployment_version` (not only reachable via `subscription`/`deployment`),
so client name / workflow name never need the subscription join for this
kind of lookup.
