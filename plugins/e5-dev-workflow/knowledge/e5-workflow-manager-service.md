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
