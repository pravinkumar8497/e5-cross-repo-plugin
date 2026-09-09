---
service: e5-workflow-manager-service
feature: credential-release-appname
status: active
related_files:
  - ReleaseCredentialTaskHandler.java
  - PodActionDecisionService.java
integrations: []
---

# credential-release-appname

Spec: `e5-feature-specs/2026-09-06-credential-release-appname/design.md`.

## 2026-09-06 — app name currently parsed from resourceName

`ReleaseCredentialTaskHandler.java` and `PodActionDecisionService.java` derive
`appName` via `resourceName.substring(0, resourceName.indexOf("_"))` (4 call
sites total). `fetchActivePodByCredential` also guards with
`resourceName.contains("_")` and throws if absent. This assumes `resourceName`
is always `<appName>_<rest>` — not guaranteed going forward. Fix: read
`appName` from `GetDeploymentDetailsByResourceNameResponse` instead (see
`e5-deployment-mgmt-service/credential-release-appname.md`).

No existing unit tests covered `ReleaseCredentialTaskHandler` or
`PodActionDecisionService` as of this date — this whole credential-release
path was under-tested.

## 2026-09-06 — implemented: read appName instead of parsing resourceName

Fix implemented on branch `defect/credential-release-appname` off
`release/v1.3.4` (commit 6624d66). Both files now read `appName` from
`GetDeploymentDetailsByResourceNameResponse` / the task's fbPayload instead of
`resourceName.substring(0, resourceName.indexOf("_"))`.

Non-obvious finding: `PodActionDecisionService.determinePodAction` was
*already* extracting an `appName` field from the task's fbPayload map
(`extractedFields.put("appName", null)`) — it just never used it for anything
beyond a log line, and kept re-deriving app name from `resourceName` instead
for the actual gRPC calls. When "a field is already extracted but unused"
shows up again, check whether the unused value is actually the fix.
