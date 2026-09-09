---
service: e5-deployment-mgmt-service
role: service
depends_on: []
consumed_by: [e5-platform-ss-credential-monitor, e5-workflow-manager-service]
features: [credential-release-appname]
integrations: []
---

# e5-deployment-mgmt-service

gRPC server (Java, no Spring, Hibernate/Postgres/Flyway, Quartz scheduler). Owns
deployment/customer/subscription/scheduling data.

## Features

- [credential-release-appname](./credential-release-appname.md) — resolving
  `appName` on credential-assignment records instead of deriving it ambiguously
  from `resourceName`/pod name.

## Environment notes (repo-wide, not feature-specific)

- **Branch convention:** this repo uses `release/vX.Y.Z` branches, not `main`, as
  the actual current line of development — `main` and even mid-range release
  branches can be stale by dozens of commits. Always check
  `git branch -a | grep release` and diff against the highest version number
  before branching for new work.
- **Local build:** `./gradlew test` works fine in this environment (no external
  dependency auth needed for this repo specifically) — unlike its downstream
  consumers, which need `e5-task-sdk-java` from a private Artifactory.
