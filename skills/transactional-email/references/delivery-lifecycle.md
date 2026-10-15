# Delivery lifecycle

## Application-owned records

Adapt this model to the existing database instead of adding a competing queue. Keep recipient addresses, tokens, and message content out of routine logs.

| Record | Relevant fields and invariants |
| --- | --- |
| Message intent/outbox | Application message ID; business dedupe key; recipient; category; template/version; minimal render data; created/expiry timestamps; next-attempt time; submission state; attempts |
| Submission attempt | Message ID; attempt ID; provider/account; provider message ID if returned; idempotency key; start/end times; outcome including `unknown`; redacted error class |
| Event inbox | Provider/account; unique provider event ID or documented deterministic dedupe key; verified receipt time; provider occurrence time; linked submission; normalized event type; minimal raw evidence with retention limit |
| Suppression | Normalized recipient key; scope; reason; source; effective time; optional expiry; audit history. Uniqueness includes scope. |

Choose an explicit recipient normalization policy. Normalize the domain; do not assume arbitrary local-part case folding, dot removal, or plus-address stripping is safe. Keep the original address for sending and display. If using hashed lookup keys, use a keyed hash with a rotation plan rather than a plain hash vulnerable to enumeration.

An ordinary submission adapter accepts a message ID, verified sender identity, recipient, subject, HTML/text, approved headers, category, and optional idempotency key. It returns provider/account/message identifiers and `accepted`, `rejected`, or `unknown`. Events normalize to delivery, temporary failure, permanent failure, complaint, and supported unsubscribe events. Preserve the provider's original type and codes.

## Queueing and submission

Commit the outbox row with the business transaction. A worker claims due rows with a lease or equivalent concurrency control. Re-check message expiry, current preferences, and suppressions at dispatch, not just at enqueue time. Prevent duplicate concurrent claims and recover abandoned leases.

Use a stable application message ID across retries. Use the provider's idempotency key only within its documented scope and retention window; a stable RFC Message-ID alone is not a deduplication guarantee. For SMTP or APIs without idempotency, describe delivery as at least once where automatic retries can duplicate a message.

Treat throttling and transient connection/server failures with capped exponential backoff and jitter, honoring Retry-After. Permanent errors stop automatic retries. Authentication and configuration failures should pause affected work and alert an operator, not suppress the recipient. If the provider owns post-acceptance retries, do not submit a new message merely because it reports a delivery deferral.

A network timeout or crash after remote acceptance creates an unknown outcome. Reconcile using provider status or a documented idempotent resubmission where possible. Otherwise use an explicit application policy: duplicate-tolerant notifications may be retried within a bounded budget; sensitive invitations/resets should preserve the same still-valid operation and token or await reconciliation. Do not silently promise exactly-once email.

Set retry budgets and message expiry according to purpose. A reset email must not be submitted after its token has expired. Keep expired/dead-letter records inspectable with redacted metadata; operators must not blindly replay obsolete identity links.

## Webhook ingestion

Use the provider's actual authentication mechanism. Where it signs bodies, verify the original raw bytes before parsing; enforce documented timestamp/replay checks and use constant-time comparison through a supported verifier. For services with a different mechanism, follow it rather than inventing an HMAC header. Require HTTPS and bounded payloads. If a provider cannot authenticate events adequately, use a secure integration alternative before letting events change suppressions or account state.

Persist a verified event durably before acknowledging it; process it asynchronously. Reject bad authentication without changing application state. Unknown event types should be recorded safely and not treated as delivery success. Deduplicate by provider account plus event ID, including batch items independently. If a provider has no stable event ID, define and test a conservative deterministic key from its documented identifiers.

Bind events to a known provider account and submission. Do not use a payload's arbitrary metadata as authorization. Allow early events to wait for reconciliation when they race the submission response. Maintain an event history and separate timestamps/flags for delivery, bounce, complaint, and unsubscribe rather than one last-write-wins state. A later delivery event cannot clear a complaint or permanent suppression.

## Suppression policy

Permanent invalid-recipient failures and complaints normally prevent future sends in their applicable scope. A provider's “permanent” rejection can instead indicate bad credentials, content, sender policy, or DNS configuration; classify the reason before suppressing an address. Temporary mailbox/network issues use bounded deferral or an expiring suppression according to policy.

Separate optional-notification opt-outs from address safety suppressions. Store the reason and originating event. A new deployment, provider change, or reordered event must not clear them. Clearing a complaint or hard-bounce suppression requires an explicit audited policy with appropriate evidence; do not auto-clear after a successful submission.

If the provider maintains a separate suppression list, reconcile its scope and precedence with the application store. Imports and failover must preserve existing suppressions. Surface disagreement to operators instead of retrying around the provider's refusal.

## Useful behavioral checks

Exercise a business transaction rollback, duplicate business event, worker crash after acceptance, provider throttling, invalid credentials, expired message, duplicate/out-of-order event, invalid signature, early event arrival, permanent recipient failure, and preference change between enqueue and dispatch. Verify externally observable state and whether a send was attempted.
