---
integration: e5-platform-ss-workflow-api <-> e5-platform-ss-webhook-service
protocol: kafka (CloudEvents v1.0) -> postgres queue -> outbound HTTP
direction: e5-platform-ss-webhook-service consumes events produced by e5-platform-ss-workflow-api
---

# e5-platform-ss-workflow-api ↔ e5-platform-ss-webhook-service

## What crosses the boundary

`e5-platform-ss-workflow-api`'s status-update pipeline
(`StatusActionResolver` → `ActionExecutor` →
`integration/notification/WebhookRequestProducer`) publishes a CloudEvents
v1.0 envelope to Kafka for each workflow status transition it needs to
notify an external client about (`ELIGIBILITY_REQUEST_CREATED`,
`ELIGIBILITY_RESPONSE_RECEIVED`, `ELIGIBILITY_RESPONSE_ANALYZED`, etc., for
the `eligibility-unified` workflow — other workflows emit their own status
vocabulary). `e5-platform-ss-webhook-service`'s `WebhookKafkaConsumer`
picks these up, persists them to its own Postgres `webhook_queue`
(`sourceService: "workflow-api"` on every row), and delivers them by HTTP
to the client's configured destination URL.

Envelope shape produced by workflow-api (and stored verbatim as
`webhook_queue.payload`):
```json
{
  "id": "evt-...",
  "data": { "entityId": "...", "entityStatus": "...", "eligibilityRequest": {...}, "eligibilityResponse": {...} },
  "time": "2026-09-17T...Z",
  "type": "e5.entity.eligibility-request.<STATUS>",
  "source": "neos/eligibility-request/<entityId>",
  "specversion": "1.0",
  "sub-tenant-id": "<tenant-uuid>",
  "datacontenttype": "application/json"
}
```

`e5-platform-ss-webhook-service` sends this envelope's fields as the
outbound HTTP body **as-is** (minus whatever keys `webhook.delivery.
payload-header-keys` promotes to headers — currently just `sub-tenant-id`
→ `X-Sub-Tenant-Id`); it does not unwrap `data`. This was investigated as a
possible cause of a client's webhooks failing (see
`knowledge/e5-platform-ss-webhook-service/outbound-body-charset-default.md`)
and ruled out — the actual cause there was character encoding, not
envelope shape — but it's worth knowing this boundary exists if a future
destination's contract turns out to expect the unwrapped `data` object at
the JSON root instead.

## Gotcha: same `entityId` can produce near-duplicate events close together

Two `ELIGIBILITY_RESPONSE_ANALYZED` events with identical `data` content
but different `id`s were observed ~3 seconds apart for the same `entityId`,
both queued to the same `destination_id`. Not yet root-caused on the
workflow-api side (out of scope of the sessions that produced this note) —
if diagnosing a client's duplicate-webhook complaints, check whether
workflow-api's status pipeline can double-publish for the same transition,
in addition to checking webhook-service's own delivery/retry behavior.

## Diagnostic path when a client's webhooks are failing

1. Check `webhook_queue` (webhook-service DB) for the client's
   `destination_id` — `last_response_code`, `attempt_count`, `status`.
2. If `DEAD_LETTERED`, the real HTTP response body is in
   `webhook_dead_letters.last_error_message`, not `webhook_queue.
   last_error_message` (which stays blank for ordinary HTTP failures by
   design — see webhook-service's `_overview.md`).
3. Compare the failing row's `payload.data` against a manually-replayed
   curl of the *exact same* content to the *exact same* destination URL —
   this is the fastest way to separate "our delivery mechanics are wrong"
   from "the destination genuinely rejects this content/status" from
   "something in transport (headers, encoding) differs between what we
   intend to send and what actually goes over the wire."
