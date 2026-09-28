---
service: e5-platform-ss-workflow-api
role: service
depends_on: [aws-secrets-manager, e5-platform-graphql, neos, kafka]
consumed_by: [e5-platform-ss-webhook-service]
features: [custom-jwt-decoder-audience-gap]
integrations: [e5-platform-ss-workflow-api--e5-platform-ss-webhook-service]
---

# e5-platform-ss-workflow-api

Java 21 / Spring Boot 3.2 / Maven. PostgreSQL (Flyway), Kafka (CloudEvents
v1.0), Auth0 JWT, GraphQL platform client, AWS S3, Resilience4j. No
per-workflow controllers — `SubmissionController` handles `POST/GET /**`;
a JWT filter resolves the request path → `Category.full_path` →
`WorkflowClientConfig` and stores it in a thread-local
`WorkflowSecurityContext`.

Emits webhook requests (as CloudEvents) that
`e5-platform-ss-webhook-service` delivers to external client endpoints — see
the integration doc.

## Features

- [custom-jwt-decoder-audience-gap](./custom-jwt-decoder-audience-gap.md) —
  the `custom` JWT decoder path's audience check was never wired to a
  per-environment env var; it silently fell back to a hardcoded QA literal
  in every environment, including prod.

## Environment notes (repo-wide, not feature-specific)

- Two JWT decoder implementations exist side by side, selected by
  `app.jwt.decoder` (`custom` | `jwks`, default `custom` per
  `application.yml`): `CustomTokenJwtDecoder` /
  `TokenValidationService` (RSA public key from Secrets Manager or
  `app.jwt.public-key-pem` for local) vs. a Nimbus/JWKS decoder wired in
  `SecurityConfig`. Only one is active per deployment based on which
  `@ConditionalOnExpression`/bean wins — when debugging JWT validation
  failures, check the stack trace for which class actually threw
  (`CustomTokenJwtDecoder` vs. the Nimbus decoder) before assuming a given
  env var is even in the code path being exercised.
- `AUTH0_AUDIENCE` is set correctly per environment in every
  `deployment-configs/*.yaml`, but historically only fed the inactive
  Nimbus/JWKS path (`SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_AUDIENCES`).
  Don't assume an env var is "wired up" just because it's present in the
  pod's env and referenced somewhere in `application.yml` — confirm it
  reaches the specific bean/decoder actually active.
