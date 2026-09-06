# e5-rf-fb-worker-core — notes

## 2026-09-06 — APP_NAME env var exists but is unused for credential events

`conf/settings.py:35` already defines `APP_NAME = os.getenv("APP_NAME",
"unknown")`, separate from `HOST_NAME = os.getenv("HOSTNAME", "")` (the pod
name, which carries a random K8s-generated suffix). Currently `APP_NAME` is
only used for OTel resource attributes (`integrations/otel/attributes.py`).

The CCS credential-fetch Kafka event is built in
`integrations/ccs/CCSServices.py` at two call sites, shape:
`"ccs_event": {"type": "FETCHED", "payload": {"pool_name":..., "pod_name":
requester_id, "fetched_time":..., "lock_id":..., "kube_namespace":...,
"resource_name":...}}`. It does not include `app_name`. This is a Python
SDK consumed by multiple teams who upgrade independently — any payload
change here must be additive/optional, never a breaking format change.

Also note: `pool_name.split("_")` in `get_login_credentials` (line ~32) is
a *different* concept (CCS resource-pool naming: `<client_name>_<application_
name>`), unrelated to the workflow-manager's resourceName-prefix parsing —
don't conflate the two when reasoning about "underscore-separated names" in
this codebase.

## 2026-09-06 — implemented: send app_name in FETCHED events

Fix implemented on branch `defect/credential-release-appname` off
`release/v1.1.12` (commit 17ba536, already the latest release branch — no
staleness issue here, unlike credential-monitor and deployment-mgmt).
Added `resolve_app_name_for_event()` in `CCSServices.py` (treats the
`"unknown"` default as "not provided") and wired it into both FETCHED
payload construction call sites (the primary attempt and the retry-on-HTTP-
error path — this file constructs the same payload dict literal twice,
watch for both when touching this event shape again).

**Local test limitation:** could not run pytest here — `tests/integrations/
conftest.py` imports `moto`, and `CCSServices.py` itself imports `robot`,
`confluent_kafka`, `jwt`, none of which are installed and this environment
has no network access to `pip install` them. Verified via `ast.parse()`
syntax check + manual review only. If this environment gains those deps
later, `tests/integrations/test_ccs_services.py` covers the new function.
