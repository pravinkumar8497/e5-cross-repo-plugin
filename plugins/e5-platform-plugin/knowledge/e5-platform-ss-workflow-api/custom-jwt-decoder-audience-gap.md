---
service: e5-platform-ss-workflow-api
feature: custom-jwt-decoder-audience-gap
status: fixed
related_files:
  - src/main/java/com/company/workflow/infrastructure/filter/TokenValidationService.java
integrations: []
---

# custom-jwt-decoder-audience-gap

## 2026-09-11 — prod 401s: "API key is missing or invalid"

Every call to `/v2/workflows/submit-eligibility` in `platform-subsystem-prod`
was rejected. JWT filter logs showed
`IncorrectClaimException: Missing expected 'https://api-qa.e5.ai/wf/v2' value
in 'aud' claim [https://api.e5.ai/wf/v2]` — i.e. the app was validating
against the **QA** audience while running in **prod**, and prod tokens
correctly carried the prod audience.

Root cause: `TokenValidationService.java:40` —
```java
@Value("${app.jwt.expected-audience:https://api-qa.e5.ai/wf/v2}") String expectedAudience,
```
`app.jwt.expected-audience` was never set in `application.yml` or any of the
5 `deployment-configs/*.yaml` files, so every environment silently fell back
to the hardcoded QA literal. Meanwhile `AUTH0_AUDIENCE` *was* set correctly
per environment (confirmed via `kubectl describe pod` on the prod pod:
`AUTH0_AUDIENCE=https://api.e5.ai/wf/v2`), but it only fed the **inactive**
Nimbus/JWKS decoder path in `SecurityConfig`
(`SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_AUDIENCES: ${AUTH0_AUDIENCE}` in
`application.yml`) — the `custom` decoder (`app.jwt.decoder=custom`, the
active one, confirmed by the exception stack trace running through
`CustomTokenJwtDecoder` → `TokenValidationService`) never read it.

Fix (one line, no deployment-config changes needed since `AUTH0_AUDIENCE`
was already correct everywhere):
```java
@Value("${app.jwt.expected-audience:${AUTH0_AUDIENCE:https://api-qa.e5.ai/wf/v2}}") String expectedAudience,
```

The `${AUTH0_AUDIENCE}` placeholder-in-placeholder pattern already had
precedent in this codebase (`application.yml:69`,
`audiences: ${AUTH0_AUDIENCE}`), so this wasn't a new mechanism — just
extending an existing, proven one to the decoder path that actually needed
it.

**Generalizable lesson:** when two implementations of the same concern exist
side by side (two JWT decoders here) and only one is active per environment,
an env var can look fully wired (present in the pod, referenced in YAML)
while actually feeding the *other*, inactive implementation. Trace the
active code path (stack trace, `@ConditionalOnExpression`) before trusting
that a config value is reaching the code that needs it.
