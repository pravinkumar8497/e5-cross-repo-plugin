---
service: e5-platform-ss-webhook-service
role: service
depends_on: [kafka, postgres, redis, aws-secrets-manager]
consumed_by: []
features: [outbound-body-charset-default, unwired-retry-eligibility-checker, fifo-per-destination-delivery]
integrations: [e5-platform-ss-workflow-api--e5-platform-ss-webhook-service]
---

# e5-platform-ss-webhook-service

Reliable HTTP webhook delivery platform. Consumes CloudEvents from Kafka,
persists to a Postgres `webhook_queue`, delivers via HTTP with
retry/backoff, per-host circuit-breaking, and dead-letters on exhaustion.

Flow: Kafka → `WebhookKafkaConsumer` → `WebhookIngestionService` (validate,
idempotency, persist) → Postgres queue → `WebhookQueuePoller` (100ms poll,
`SKIP LOCKED`, virtual threads) → `WebhookHttpClient` → retry or
dead-letter.

## Features

- [outbound-body-charset-default](./outbound-body-charset-default.md) —
  outbound bodies with no explicit charset in `Content-Type` were silently
  sent as ISO-8859-1, corrupting non-ASCII characters. Fixed.
- [unwired-retry-eligibility-checker](./unwired-retry-eligibility-checker.md)
  — a correctly-implemented `RetryEligibilityChecker` exists but is never
  called; the poller retries any non-2xx/429/5xx response regardless of a
  destination's own `retryOnStatusCodes` allow-list. **Known, not fixed** —
  a fix was written and then reverted at the user's request; see the doc for
  why it's still correct and safe to reintroduce later.
- [fifo-per-destination-delivery](./fifo-per-destination-delivery.md) — how
  strict per-`destination_id` FIFO plus per-host circuit breaking interact,
  and where to actually find a delivery's real HTTP response body when
  diagnosing a failure.

## Environment notes (repo-wide, not feature-specific)

- **Where the real response body lives**: `webhook_queue.last_error_message`
  stays blank for ordinary HTTP failures (4xx/5xx) — by design,
  `DeliveryResult.retryable(code, body, duration)` puts the response body
  into `responseBody()`, not `errorMessage()` (that field is reserved for
  exceptions: timeouts, connection errors). The actual response body only
  gets persisted once a webhook is exhausted, into
  `webhook_dead_letters.last_error_message`
  (`RetryExhaustionHandler.applyDeliveryResult`, falls back to
  `result.responseBody()` when `errorMessage()` is null — which it always is
  for a plain HTTP error response). When diagnosing a dead-lettered webhook,
  query `webhook_dead_letters`, not `webhook_queue`.
- Circuit breaker key is **hostname only**, not per-destination-id/URL-hash
  — traffic to unrelated destinations on the same host can trip/share a
  breaker.
- `RetryConfig`/`retryOnStatusCodes` is stored per-webhook (a JSON snapshot
  taken at ingestion time, in `webhook_queue.retry_config`), so different
  clients/destinations can and do carry different retry policies
  side-by-side in the same table (e.g. one client's config allows retrying
  `429`, another's allows `401` instead — check the actual row's
  `retry_config`, don't assume a single global policy).
