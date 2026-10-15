---
name: transactional-email
description: Implement application transactional email, including password resets, invitations, notifications, provider integration, delivery events, and suppression handling. Use for application email workflows rather than DNS-only diagnosis or bulk campaigns.
---

# Transactional email

Build the requested email flow into the existing application's authentication, database, jobs, and test conventions. Preserve an existing sending backend unless changing it is part of the task. This skill does not require a diagnostic service or another installed skill.

## Establish the delivery boundary

Inspect current auth/token handling, sender configuration, database transactions, workers, provider clients, and event handlers before adding parallel infrastructure. Determine which messages are essential transactional mail and which are optional notifications requiring preferences or unsubscribe behavior.

When choosing a backend, read [provider selection](references/provider-selection.md). Compare viable services against the actual deployment, region, volume, events, and operations needs. APIs and SMTP are both valid. Do not select a vendor because it appears in an example. Check current official documentation for the chosen SDK/API, event verification, idempotency behavior, suppression scope, and limits.

## Implement the message lifecycle

Use [delivery lifecycle](references/delivery-lifecycle.md) for the outbox, submission interface, retries, event ingestion, and suppressions. Preserve these invariants:

- Persist the business change and its email intent atomically when they share a database. Send from a durable worker, with an application message ID and deduplication policy.
- Check suppressions immediately before submission. Record provider acceptance separately from recipient-server delivery; neither proves inbox placement or that the recipient read the message.
- Handle transient failure, permanent failure, and unknown submission outcomes separately. A timeout after submission may already have sent the email.
- Authenticate provider events using its documented mechanism, deduplicate them, and tolerate reordering. Do not trust a tenant or user ID supplied in an unauthenticated payload.
- Keep provider-specific logic in an adapter; use capability checks when behavior differs. Do not claim exactly-once delivery through a generic abstraction.

## Implement the requested flow

Read [identity flows](references/identity-flows.md) for password resets and invitations. Preserve the existing identity system when it already provides secure tokens. Tokens need expiry, purpose binding, single use under concurrency, and bounded request/redemption rates. Reset responses must not disclose whether an account exists.

Notifications should carry a stable business-event ID for deduplication and the minimum necessary personal information. Respect per-category preferences and quiet hours where applicable. Avoid embedding sensitive data or long-lived bearer links in email. Render escaped HTML and a useful plain-text alternative; protect headers from injection. Build absolute HTTPS links from trusted application configuration, not an incoming Host header.

Classify permanent recipient failures and complaints into durable suppression records. Handle temporary delivery deferrals with a bounded policy. Allow preferences to control optional notifications without accidentally silencing security-critical messages. A complaint or permanent address failure still needs a safe account-recovery path; do not bypass suppression by switching providers.

## Verify and deliver

Use a sandbox or mocked backend for automated checks; send real test email only within the user's authorized scope. Exercise token replay and expiry, worker retries and ambiguous acceptance, duplicate/reordered webhook events, signature rejection, and suppressed recipients. Reuse the application's existing meaningful tests rather than adding assertions about implementation wording.

Document required environment variables without values, sender/domain verification, worker setup, event endpoint configuration, retry limits, and how to inspect a failed message. Report what works, what was verified, and any remaining provider or deployment setup. Missing production credentials should not prevent completing and testing local integration.
