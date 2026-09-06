# e5-workflow-manager-service — notes

## 2026-09-06 — app name currently parsed from resourceName

`ReleaseCredentialTaskHandler.java` and `PodActionDecisionService.java`
derive `appName` via `resourceName.substring(0, resourceName.indexOf("_"))`
(4 call sites total). `fetchActivePodByCredential` also guards with
`resourceName.contains("_")` and throws if absent. This assumes
`resourceName` is always `<appName>_<rest>` — not guaranteed going forward.
See `e5-feature-specs/2026-09-06-credential-release-appname/design.md` for
the fix: read `appName` from `GetDeploymentDetailsByResourceNameResponse`
instead (see `e5-deployment-mgmt-service.md` notes).

No existing unit tests cover `ReleaseCredentialTaskHandler` or
`PodActionDecisionService` as of this date — this whole credential-release
path is under-tested; add tests alongside any change here.

## 2026-09-06 — implemented: read appName instead of parsing resourceName

Fix implemented on branch `defect/credential-release-appname` off
`release/v1.3.4` (commit 6624d66). Both files now read `appName` from
`GetDeploymentDetailsByResourceNameResponse` / the task's fbPayload instead
of `resourceName.substring(0, resourceName.indexOf("_"))`.

Non-obvious finding: `PodActionDecisionService.determinePodAction` was
*already* extracting an `appName` field from the task's fbPayload map
(`extractedFields.put("appName", null)`) — it just never used it for
anything beyond a log line, and kept re-deriving app name from
`resourceName` instead for the actual gRPC calls. When "a field is already
extracted but unused" shows up again, check whether the unused value is
actually the fix.

**Local build limitation:** this repo needs `e5-task-sdk-java` from a
private Artifactory (401 without credentials) — could not run
`./gradlew build`/`test` in this environment. Verified by manual review
only. Also: `gradle/wrapper/gradle-wrapper.jar` is gitignored here (unlike
e5-deployment-mgmt-service, where it's committed) — copying it in from
another repo with the same Gradle version (8.8) gets `./gradlew` running
locally, but doesn't fix the Artifactory auth issue.
`release/v1.3.4` was already the latest release branch at time of writing.
