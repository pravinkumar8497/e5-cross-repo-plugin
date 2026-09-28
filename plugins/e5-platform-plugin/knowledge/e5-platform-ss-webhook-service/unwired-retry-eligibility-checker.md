---
service: e5-platform-ss-webhook-service
feature: unwired-retry-eligibility-checker
status: known-not-fixed
related_files:
  - src/main/java/com/webhook/service/service/retry/RetryEligibilityChecker.java
  - src/main/java/com/webhook/service/service/core/WebhookQueuePoller.java
integrations: []
---

# unwired-retry-eligibility-checker

## 2026-09-17 — found while diagnosing the same client's 400s

While investigating the `outbound-body-charset-default` incident, one
client's `webhook_queue` rows had `retry_config.retryOnStatusCodes: [401,
500, 502, 503, 504]` — deliberately excluding `400` — yet `attempt_count`
climbed to 5 (the configured max) on a `400` response before dead-lettering.
By that destination's own configured policy, a `400` should never have
been retried at all.

`RetryEligibilityChecker.isEligibleForRetry()` exists and correctly
implements exactly this check:
```java
List<Integer> retryStatusCodes = config.getEffectiveRetryOnStatusCodes();
return retryStatusCodes.contains(result.responseCode());
```
`grep -rn "RetryEligibilityChecker"` across `src/main` only ever finds its
own class declaration — it is never instantiated/called anywhere.
`WebhookQueuePoller.handleRetryable()` retries any result whose
`ResponseHandler`-assigned outcome is `RETRYABLE`, and `ResponseHandler.
mapToResult()` marks *every* non-2xx/429/5xx status code as `RETRYABLE`
unconditionally (`ResponseHandler.java:65-66`, comment says "1xx, 3xx" but
the code applies to all other codes including 4xx) — the per-destination
allow-list is never consulted.

A fix was written (inject `RetryEligibilityChecker` into
`WebhookQueuePoller`, gate `handleRetryable()`'s retry-scheduling on
`isEligibleForRetry()`, else route straight to `handleExhaustion()`) and
verified to compile. **It was reverted at the user's explicit request in
this session** — not because it was wrong, but scope/timing wasn't wanted
then. It remains a real, low-risk fix worth reintroducing: it only stops
retrying status codes a destination's own config already excludes, so it
cannot change behavior for any destination/status combination that's
currently working as configured.

If picked back up, the change is exactly:
```java
// constructor: add RetryEligibilityChecker retryEligibilityChecker field + param, assign it
// in handleRetryable():
if (currentAttempt >= retryConfig.getEffectiveMaxRetries()
        || !retryEligibilityChecker.isEligibleForRetry(result, retryConfig, currentAttempt)) {
    handleExhaustion(webhook, result, currentAttempt);
    return;
}
```

**Generalizable lesson:** a class existing and being correctly implemented
is not evidence it's wired into the runtime path — always `grep` for actual
call sites, especially for anything named `*Checker`/`*Validator`/
`*Eligibility*` that reads like it should be gating a decision.
