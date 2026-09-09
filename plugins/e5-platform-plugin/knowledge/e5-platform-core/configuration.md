---
service: e5-platform-core
feature: configuration
status: active
related_files:
  - src/main/java/com/e5/platform/core/config/ConfigsLoader.java
  - src/main/java/com/e5/platform/core/config/EngineConfig.java
  - src/main/java/com/e5/platform/core/config/WorkerConfig.java
integrations: []
---

# Configuration (ConfigsLoader / config schema binding)

## 2026-09-09 — Lombok/JavaBean gotcha: two-capital-letter field names break SnakeYAML/Introspector binding

A config field named with a lowercase-then-two-uppercase pattern (`uIActionMappings`)
breaks: Lombok generates `getUIActionMappings()` (capitalizes only the first letter),
but `java.beans.Introspector.decapitalize` special-cases two-leading-caps and does
**not** lowercase the first letter back — so the JavaBean property resolves to
`UIActionMappings`, not `uIActionMappings`. SnakeYAML (used by `ConfigsLoader`) binds
by that Introspector-derived property name, so a YAML key of `uIActionMappings` fails
with `Unable to find property 'uIActionMappings'`. Renamed to `uiActionMappings`
everywhere (Java field, `EngineConfig.schema.json`/`WorkerConfig.schema.json`, config
YAML) to sidestep it. Watch for this pattern (`xYField`) on any new config field.
