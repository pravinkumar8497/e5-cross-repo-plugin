---
service: e5-platform-ss-webhook-service
feature: fifo-per-destination-delivery
status: active
related_files:
  - src/main/java/com/webhook/service/service/core/WebhookQueuePoller.java
  - src/main/java/com/webhook/service/service/queue/WebhookQueueService.java
integrations: []
---

# fifo-per-destination-delivery

## Design (from this repo's CLAUDE.md, confirmed while diagnosing 400s)

`WebhookQueuePoller` polls with `DISTINCT ON (destination_id) ORDER BY
created_at ASC` — strict FIFO **per `destination_id`**, one in-flight
delivery attempt selected per poll cycle per destination. Circuit breaking
is keyed by **hostname only** (`HostExtractor.extractHost`), not
per-destination-id/hash, so unrelated destinations sharing a host can
affect each other's circuit state.

## Consequence worth knowing before debugging "why are these 3 events all
stuck/failing together"

If the oldest pending webhook for a `destination_id` fails and enters
backoff, it does not strictly block every later item behind it forever —
the poller selects whichever due item is oldest at each 100ms tick, so an
item still in its backoff window can be passed over in favor of a
later-created item that's already due. In practice this means multiple
events for the same destination can end up retrying somewhat
interleaved rather than strictly sequentially, which can look confusing
when reading `webhook_queue.last_attempt_at` timestamps chronologically
against `created_at` order — don't assume strict one-at-a-time completion
order when reconstructing a timeline from these columns.

## Where to actually find out *why* something failed

See `_overview.md`'s "Where the real response body lives" note —
`webhook_queue.last_error_message` is not it for ordinary HTTP failures;
`webhook_dead_letters.last_error_message` is, once a webhook is exhausted.
This tripped up a chunk of a debugging session because the obvious column
to check (`webhook_queue.last_error_message`) is blank by design for the
most common failure case (a plain non-2xx HTTP response), not because of a
bug.
