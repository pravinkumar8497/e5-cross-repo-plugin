---
service: e5-rf-fb-worker-core
role: library
depends_on: []
consumed_by: [e5-platform-ss-credential-monitor]
features: [credential-release-appname]
integrations: []
---

# e5-rf-fb-worker-core

Python SDK/worker consumed by multiple teams who upgrade independently — any
payload/event shape change here must be additive/optional, never a breaking
format change. Publishes `CredentialFetchedEvent`/`CredentialReleasedEvent` to
Kafka (`integrations/ccs/CCSServices.py`).

## Features

- [credential-release-appname](./credential-release-appname.md) — adding
  `app_name` to the FETCHED credential Kafka event.

## Environment notes (repo-wide, not feature-specific)

- `conf/settings.py:35` defines `APP_NAME = os.getenv("APP_NAME", "unknown")`,
  separate from `HOST_NAME = os.getenv("HOSTNAME", "")` (the pod name, which
  carries a random K8s-generated suffix).
- `pool_name.split("_")` in `get_login_credentials` is a *different* concept
  (CCS resource-pool naming: `<client_name>_<application_name>`), unrelated to
  the workflow-manager's resourceName-prefix parsing — don't conflate the two
  when reasoning about "underscore-separated names" in this codebase.
- **Local test limitation:** `tests/integrations/conftest.py` imports `moto`,
  and `CCSServices.py` itself imports `robot`, `confluent_kafka`, `jwt` — none
  installed in this environment and no network access to `pip install` them.
  Verify via `ast.parse()` syntax check + manual review only until this
  environment gains those deps.
