# Tenant sending lifecycle

## Data and authorization

Adapt the existing schema. A useful model includes tenants, sender identities, provider-account bindings, message intents, submission attempts, event inbox records, scoped suppressions, usage reservations, and audit events. Each child record includes `tenant_id`; use composite keys/foreign keys or equivalent constraints to make foreign-tenant references impossible. Scope unique keys and dedupe keys deliberately rather than assuming IDs are globally safe.

Authenticate tenant API keys with explicit send/read/admin scopes and revocation. Resolve tenant identity from server-side authentication, not a request field. Apply the same authorization in workers, internal endpoints, exports, dashboards, and support tools. Provider credentials belong in a secret manager; never return them to customers or log them. Redact secrets and recipient data in audit records while retaining actor and resource identity.

## Domain onboarding

1. Create a pending sender identity and a random, tenant-bound, expiring DNS ownership challenge at a domain-specific name. Alternatively use an existing trusted domain-ownership mechanism. A pre-existing SPF/DKIM record or shared provider verification result alone is insufficient proof of this tenant's ownership.
2. Require the challenge, then configure provider verification and DKIM. Show the exact names, values, and record types. Tell customers to merge legitimate SPF sources rather than add a second SPF record. Explain whether the provider uses a custom envelope-from domain and which DMARC alignment path will work.
3. Keep ownership verification, provider verification, DNS health, and active sending authorization as separate states. Recheck asynchronous provider readiness before activation. Begin DMARC monitoring where appropriate; do not impose enforcement before all legitimate streams are inventoried.
4. Reject ambiguous domain claims. By default, an exact sending domain has one active tenant owner. Parent/subdomain overlap requires an explicit product policy; never grant a subdomain owner control of the parent or all siblings. Shared platform domains require platform-controlled per-tenant identities and appropriate abuse controls.
5. Recheck ownership on a bounded schedule and suspicious changes, accounting for DNS TTL and temporary resolver errors. Avoid immediate suspension on a single timeout; sustained loss or a conflicting claim enters a documented grace/reverification process. Revoke stale verification tokens on transfer and require new proof before reassignment.

Never automatically activate a customer domain because another tenant verified it in a shared provider account. Domain transfer/offboarding must block old queued sends and invalidate stale provider/domain mappings before a new owner can use it.

## Dispatch and fair usage

Write an email intent with the business transaction. Bind it to tenant, sender identity, category, and a stable application message ID. At dispatch, verify the current tenant status and ownership, sender readiness, expiry, recipient policy, and suppressions.

Reserve quota atomically before submission; concurrent workers must not exceed the limit. Track reserved, submitted, and billable usage separately. Define how failures, unknown outcomes, and lease recovery affect reservation release so a crash neither leaks quota forever nor frees already-spent capacity. Apply per-tenant rate, daily volume, and message-size limits plus account-level limits. Set smaller probationary limits for newly approved senders.

Use tenant-aware scheduling or per-tenant queues with bounded concurrency and fair scheduling. One high-volume customer must not consume all worker slots or provider capacity. Separate abuse suspension from transient throttling. Tenant suspension is checked at dispatch and blocks stale jobs immediately.

Use an application-owned submission ledger to map `(provider, provider_account, provider_message_id)` to the tenant and message. Provider IDs may be unique only within an account. Do not derive routing from email addresses or arbitrary event metadata. Retries preserve message identity; a timeout after submission may already have sent the message. Reconcile or use the provider's documented idempotency mechanism before resubmission; otherwise document the duplicate risk and bound retries by message purpose.

## Events and suppressions

Authenticate events using the provider's documented mechanism before persisting actionable state. Verify raw bytes for body-signed events, enforce documented replay checks, deduplicate event IDs within provider/account, acknowledge only after durable ingestion, and tolerate reordering and early arrival before the send response. Retry unresolved known-account events through reconciliation; quarantine unexplained ownership rather than guessing from metadata.

Classify failures by reason: bad recipient, complaint, unsubscribe, transient delivery issue, sender configuration, provider outage, or abuse suspension. Configuration and authentication errors do not make a recipient invalid. A delivery event cannot clear a complaint or hard-bounce suppression.

| Suppression scope | Policy |
| --- | --- |
| Tenant/category | Customer-specific consent or optional notification preferences; cannot affect another tenant |
| Tenant/address | Tenant complaint or permanent failure, subject to the chosen safety policy |
| Platform/address or platform/sender | Explicit platform safety policy; administrative access only; tenants cannot bypass it |
| Provider/account | Respect actual provider suppression behavior; document when it is broader than the application boundary |

Do not globally suppress a recipient merely because they opted out from an unrelated tenant. Conversely, do not route around a provider or platform safety suppression. If the provider only offers a shared suppression store and tenant independence is required, choose stronger account separation or another backend. Reconcile suppressions before failover or migration, including evidence and effective scope. Never auto-clear them during a tenant transfer.

## Abuse and reputation

Define the permitted message classes and consent requirements before onboarding arbitrary customer content. Control arbitrary recipients/content, invitation abuse, imported lists, and signup automation. Bulk mail needs consent tracking, appropriate unsubscribe handling, and current mailbox-provider requirements in addition to this pipeline.

Monitor accepted volume, queue age, permanent-failure and complaint rates, verification failures, and quota denials per tenant. Use defined observation windows and minimum sample sizes before automatic suspension; provider feedback coverage is incomplete. Provide operator review, a tenant-visible explanation, and a documented restoration process.

Credential separation is not reputation separation. Document shared IP pools, DKIM domains, account limits, and provider-wide suspension risk. Higher-risk or contractually isolated tenants may need separate accounts/pools. Dedicated IPs require a justified volume and warmup plan. Failover must not bypass suspension or weaken tenant authorization.

## Offboarding and operational checks

Disable tenant dispatch, revoke API/provider credentials, cancel or tombstone queued messages, stop new quota reservations, retain necessary event-to-tenant mappings for late events, and apply the retention/deletion policy. Releasing a domain must not let its next owner access prior events or content.

Tests should observe send attempts and state changes for foreign sender IDs, foreign message IDs, guessed domain IDs, forged tenant metadata, concurrent quota reservations, duplicate/reordered/early events, queue fairness, tenant suspension during retry, provider-account ID collisions, suppression leakage, and domain reassignment with old jobs pending.
