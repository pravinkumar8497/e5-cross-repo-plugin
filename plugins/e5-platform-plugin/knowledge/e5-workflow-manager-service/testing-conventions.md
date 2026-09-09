---
service: e5-workflow-manager-service
feature: testing-conventions
status: active
related_files:
  - EnableDeploymentServiceTest.java
integrations: []
---

# testing-conventions

## 2026-09-06 — MockedStatic<ProtoFactory>, and a real .isEmpty()/.isBlank() bug

This repo's established pattern for testing code that calls
`ProtoFactory.getStub(ServiceType.X, StubClass.class)` is `@Mock` on the stub
field + `@ExtendWith(MockitoExtension.class)` + wrapping the call in `try
(MockedStatic<ProtoFactory> m = mockStatic(ProtoFactory.class)) {...}` — see
`EnableDeploymentServiceTest` for the reference example. This repo has no
convention for testing *private* methods; reflection
(`getDeclaredMethod(...).setAccessible(true)`) is a reasonable one-off choice
when the public entry point pulls in unrelated heavy static dependencies
(here, `ReleaseCredentialTaskHandler.performCredentialRelease` would also
require mocking `ResourceNegotiatorUtil`'s HTTP-backed statics just to reach
the method actually worth testing).

Writing a blank-string test for the `appName` validation (added during the
credential-release-appname feature) surfaced a real bug:
`PodActionDecisionService` and `ReleaseCredentialTaskHandler` both validated
`appName` with `.isEmpty()`, which doesn't catch whitespace-only values.
Fixed to `.isBlank()` — but `resourceName`/`kubeNamespace` checks elsewhere in
these same files were deliberately left as `.isEmpty()` (not touched, out of
scope). If validating a new field with `.isEmpty()`, write the blank-string
test case first — it would have caught this immediately.
