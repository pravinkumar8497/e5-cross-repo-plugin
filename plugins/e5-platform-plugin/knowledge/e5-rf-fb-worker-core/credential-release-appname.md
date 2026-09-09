---
service: e5-rf-fb-worker-core
feature: credential-release-appname
status: active
related_files:
  - integrations/ccs/CCSServices.py
  - tests/integrations/test_ccs_services.py
integrations: []
---

# credential-release-appname

Spec: `e5-feature-specs/2026-09-06-credential-release-appname/design.md`.

## 2026-09-06 — APP_NAME env var exists but was unused for credential events

The CCS credential-fetch Kafka event is built in `integrations/ccs/CCSServices.py`
at two call sites, shape: `"ccs_event": {"type": "FETCHED", "payload":
{"pool_name":..., "pod_name": requester_id, "fetched_time":..., "lock_id":...,
"kube_namespace":..., "resource_name":...}}`. It did not include `app_name`.

## 2026-09-06 — implemented: send app_name in FETCHED events

Fix implemented on branch `defect/credential-release-appname` off
`release/v1.1.12` (commit 17ba536, already the latest release branch — no
staleness issue here, unlike credential-monitor and deployment-mgmt). Added
`resolve_app_name_for_event()` in `CCSServices.py` (treats the `"unknown"`
default as "not provided") and wired it into both FETCHED payload
construction call sites (the primary attempt and the retry-on-HTTP-error
path — this file constructs the same payload dict literal twice, watch for
both when touching this event shape again).

`tests/integrations/test_ccs_services.py` covers the new function (see
`_overview.md` for why it couldn't be run in this environment).
