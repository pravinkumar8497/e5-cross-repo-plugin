---
service: e5-platform-ss-webhook-service
feature: outbound-body-charset-default
status: fixed
related_files:
  - src/main/java/com/webhook/service/service/delivery/WebhookRequestBuilder.java
integrations: [e5-platform-ss-workflow-api--e5-platform-ss-webhook-service]
---

# outbound-body-charset-default

## 2026-09-17 — demo: one client's webhooks failing with HTTP 400, others fine

Symptom: for one client ("Oscorp Health" / destination `dest-healthcare-ERM-
Stable`), 3 of 5 queued webhooks dead-lettered with `last_response_code =
400`; 2 succeeded. Same destination URL, same auth, same tenant header on
every call — so first suspicion was payload shape (CloudEvents envelope vs.
flat body) or the `entityStatus` value. Both were ruled out by direct
`curl`/Postman replay of the *exact* failing payload (envelope-wrapped,
flat, either `entityStatus` value) against the real endpoint — all
succeeded manually. Only the traffic actually sent by this service failed.

That isolated it to something the manual curl wasn't reproducing: the
**successful** payloads had no non-ASCII characters; the **failing** ones
all had a populated `eligibilityResponse.payerDetails.planName` containing
`®` (U+00AE, "AARP® Medicare Advantage...").

Root cause: `WebhookRequestBuilder.setBody()` built the outbound
`StringEntity` from `ContentType.parse(config.getEffectiveContentType())`.
No destination in this environment configures an explicit charset
(`DestinationConfig.DEFAULT_CONTENT_TYPE = "application/json"`, no
`;charset=`), and Apache HttpClient5's `StringEntity` falls back to
**ISO-8859-1**, not UTF-8, when the `ContentType` carries no charset. Any
non-ASCII character in the payload was therefore sent as the wrong byte
sequence — a receiving server parsing the body as UTF-8 JSON sees invalid
bytes and rejects with 400. ASCII-only payloads are unaffected (identical
under both charsets), which is exactly why only some events for this client
failed and no other client/destination was affected — it's the one whose
data happened to contain a non-ASCII character, not a client-specific
config problem.

Fix, in `setBody()`:
```java
String contentType = config.getEffectiveContentType();
ContentType parsedContentType = ContentType.parse(contentType);
if (parsedContentType.getCharset() == null) {
    // ContentType.parse() with no charset falls back to ISO-8859-1 in StringEntity,
    // corrupting non-ASCII characters (e.g. "®") on the wire. Default to UTF-8.
    parsedContentType = parsedContentType.withCharset(java.nio.charset.StandardCharsets.UTF_8);
}
StringEntity entity = new StringEntity(body, parsedContentType);
```
Only fires when a destination's configured `Content-Type` has no charset —
any destination that already specifies one keeps its own behavior.
Zero-impact on ASCII-only traffic (the overwhelming majority), so this was
safe to fix globally rather than scoping it to one destination.

**Generalizable lesson:** when "same endpoint, same headers, same auth,
only some payloads fail" and manual curl replay of the *exact* failing
body succeeds every time, stop suspecting the payload/shape/status and
start suspecting the transport layer between "what the code intends to
send" and "what actually goes over the wire" — serialization charset is an
easy one to miss because it's invisible in code review (nothing looks wrong
reading `ContentType.parse("application/json")`) and only bites on
non-ASCII input.
