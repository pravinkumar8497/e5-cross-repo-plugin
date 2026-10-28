---
service: e5-platform-ss-interface
role: service
depends_on: [e5-deployment-mgmt-service, e5-task-sdk-java]
consumed_by: []
features: []
integrations: [e5-deployment-mgmt-service--e5-platform-ss-interface, e5-platform-ss-interface--e5-workflow-manager-service]
platform_features: [priority-aware-credential-sharing]
---

# e5-platform-ss-interface

Spring Boot GraphQL gateway (Java 21, `spring-boot-starter-graphql` on
`graphql-java`). Client-facing API surface — resolves GraphQL
queries/mutations by calling into `e5-deployment-mgmt-service` over gRPC and
publishing task messages to `e5-task-sdk-java`'s `TaskTransport`
envelope/Kafka routing for downstream services (task-manager → AKST →
workflow-manager-service) to pick up. Not consumed by any other repo in this
set — it's the outermost layer.

## Platform features (cross-repo — see `features/`, not duplicated here)

- [priority-aware-credential-sharing](../../features/priority-aware-credential-sharing.md) —
  this repo's piece: `scaleAppWorker` mutation extended to accept
  `groupName`/`groupCount` (scale a whole credential-sharing group)
  alongside the existing per-app `deploymentVersionId`/`appName`/
  `minReplicas`/`maxReplicas` shape. Also spans `e5-platform-core`,
  `e5-deployment-mgmt-service`, `e5-workflow-manager-service`.

## Environment notes (repo-wide, not feature-specific)

- **`net.devh:grpc-client-spring-boot-starter:3.1.0.RELEASE` is mismatched
  with the rest of this Spring Boot 3.3.4 stack** — it's an old starter that
  pulls its own transitive `io.grpc:*` artifacts (`grpc-core`,
  `grpc-netty-shaded`, `grpc-inprocess` at `1.63.0`) alongside the `1.68.1`
  declared directly in `build.gradle`, and Gradle doesn't dedupe them —
  `NoClassDefFoundError: io/grpc/InternalGlobalInterceptors` at startup
  building the gRPC channel bean. Fixed with a `configurations.all { resolutionStrategy.force
  'io.grpc:grpc-core:1.68.1', ... }` block forcing every `io.grpc:*` artifact
  to one version. A second, same-root-cause failure surfaces right after:
  `net.devh`'s `GrpcClientMicrometerTraceAutoConfiguration` assumes a
  `micrometer-core` artifact (for `ObservationGrpcClientInterceptor`) that
  isn't actually on this classpath (only `micrometer-observation`/
  `-commons`/`-jakarta9` are) — worked around with
  `spring.autoconfigure.exclude=net.devh.boot.grpc.client.autoconfigure.GrpcClientMicrometerTraceAutoConfiguration`
  rather than adding `micrometer-core` as a new dependency. As far as could
  be determined, this app had never actually been run to a full local
  Spring Boot startup before — every earlier attempt (including automated
  ones) got stuck on Artifactory access before reaching runtime, so this
  conflict was latent and undiscovered.
- **`SecurityConfig`'s `jwtDecoder()` bean makes a real HTTP call at
  startup** (`JwtDecoders.fromOidcIssuerLocation(issuer)` fetches OIDC
  discovery metadata) — a dummy/unreachable `OAUTH_ISSUER` doesn't let the
  app start with a no-op decoder, it just trades one startup crash for
  another. For local testing with no real IdP available, the whole
  `filterChain`/`jwtDecoder` pair needs to be swapped for a `permitAll()`
  chain with no `JwtDecoder` bean at all (not just told to skip auth via a
  property) — see the commented-out original left in place in
  `SecurityConfig.java`, marked "TEMPORARY FOR LOCAL TESTING".
- **Kafka Streams needs its source topic to exist before the app finishes
  starting** — `TASK_MANAGER_OUTBOUND_TOPIC`'s consumer-side Streams
  topology throws `MissingSourceTopicException` and shuts down that one
  background thread if the topic doesn't exist yet on the broker. Non-fatal
  to the rest of the app (HTTP/GraphQL server, gRPC client, the Kafka
  *producer* path `scaleAppWorker` actually uses all come up fine regardless)
  but worth creating the topic anyway to avoid the error spam locally.
- **`@Valid` + `@Validated` method-level validation does not fire for
  `@MutationMapping` handlers** — see `features/priority-aware-credential-sharing.md`
  for the full story (a mixed-mode `scaleAppWorker` request this let through
  to production); this is the one most worth not re-discovering by trial
  and error.
- **No Flyway/DB of its own** — all persistence goes through
  `e5-deployment-mgmt-service` over gRPC; nothing to migrate or seed in this
  repo directly.
