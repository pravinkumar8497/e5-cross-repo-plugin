---
service: e5-workflow-manager-service
role: service
depends_on: [e5-deployment-mgmt-service]
consumed_by: []
features: [credential-release-appname, testing-conventions]
integrations: []
---

# e5-workflow-manager-service

Java service. Not present in the `e5-platform-kb` submodule workspace as of
2026-09-06 (referenced there as a Kafka consumer of
`e5-ss-dynamic-scaling-service`'s task queue).

## Features

- [credential-release-appname](./credential-release-appname.md) — reading
  `appName` from `e5-deployment-mgmt-service` instead of parsing it out of
  `resourceName`.
- [testing-conventions](./testing-conventions.md) — this repo's mocking
  pattern for `ProtoFactory`-based gRPC calls, and a real `.isEmpty()`/
  `.isBlank()` bug it surfaced.

## Environment notes (repo-wide, not feature-specific)

- **Local build limitation:** this repo needs `e5-task-sdk-java` from a private
  Artifactory (401 without credentials) — could not run `./gradlew build`/`test`
  in this environment. Verify by manual review only.
- `gradle/wrapper/gradle-wrapper.jar` is gitignored here (unlike
  `e5-deployment-mgmt-service`, where it's committed) — copying it in from
  another repo with the same Gradle version (8.8) gets `./gradlew` running
  locally, but doesn't fix the Artifactory auth issue.
- **Branch convention:** check `git branch -a | grep release` and use the
  latest release branch as the base for new work (`release/v1.3.4` was current
  at time of writing).
