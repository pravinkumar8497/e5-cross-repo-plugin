# e5-deployment-mgmt-service — notes

## 2026-09-06 — resourceName -> appName has no reliable existing join

`GetDeploymentDetailsByResourceNameUseCase` (native SQL in
`DeploymentStateRepositoryImpl.findDeploymentDetailsByResourceName`) joins
`credential_assignment_registry` -> `deployment_state` -> `deployment_version`
-> `customer`/`workflow_sku`/`workflow_variant`. It resolves a
`deployment_version_id` but **not** an application. `fb_worker_pod` links
`deployment_version_id` -> `application_id`, but that FK is one-to-many (a
single deployment version can host multiple applications' worker pods), so
joining through it from resourceName alone is ambiguous — do not do this to
get a per-credential app name.

`credential_assignment_registry` already stores both `resource_name` and
`pod_name` on the same row (set in `addAssignedCredentialToStore` /
`updateReleaseStatusToStore`, both keyed by the `CredentialAssignmentRegistry`
entity). This is the right row to add an `app_name` column to — see
`e5-feature-specs/2026-09-06-credential-release-appname/design.md`.

## 2026-09-06 — fb-worker pod naming convention

Real pod names (from `e5-workflow-manager-service`'s test fixtures) follow
`<appName>-fb-worker-<version>-<NN>-<replicaset-hash>-<pod-hash>`, e.g.
`sample-fb-worker-v1-01-5c89d896dd-zvhsg`. The literal `-fb-worker` marker is
already a platform convention (see `FB_WORKER_POD_PREFIX` in
`e5-platform-ss-credential-monitor`'s `KubernetesPodsMonitor`), so
`podName.substring(0, podName.indexOf("-fb-worker"))` is a safe way to
recover the app name from a pod name when nothing better is available.
Treat this as a transitional fallback only (see design doc's "Known
limitation").

## 2026-09-06 — implemented: app_name resolution + release branch conventions

Fix implemented on branch `defect/credential-release-appname` (commit
1dd0d39): `credential_assignment_registry.app_name` column added, resolved
in `DeploymentManagementServiceImpl.addAssignedCredentialToStore` (use
`PodCredentialAssignment.appName` if present, else
`StringUtil.deriveAppNameFromPodName(podName)`), exposed via
`GetDeploymentDetailsByResourceNameResponse.appName`.

**Branch convention:** this repo uses `release/vX.Y.Z` branches, not
`main`, as the actual current line of development — `main` and even
mid-range release branches can be stale by dozens of commits. Always check
`git branch -a | grep release` and diff against the highest version number
before branching for new work (at time of writing, `release/v1.8.0` was
current; `release/v1.6.10` was 2 releases behind but happened to have no
diff in the credential-release code paths — don't assume that's always
true).

**Local build limitation:** `./gradlew test` works fine in this
environment (no external dependency auth needed for this repo specifically).

