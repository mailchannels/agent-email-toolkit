---
name: multi-tenant-email
description: Build SaaS customer email sending with tenant isolation, customer domain onboarding, per-tenant quotas, event routing, suppressions, and abuse controls. Use when customers send through an application, beyond the application's own transactional mail.
---

# Multi-tenant email

Implement customer sending as a tenant-scoped product capability. Identify the application's existing tenancy model, provider, jobs, event handlers, and operator controls. This skill is self-contained and does not require another installed skill or a MailChannels account.

## Choose boundaries before a backend

Determine whether customers send platform-authored transactional messages, their own transactional content, or bulk campaigns. Identify expected tenant count, volumes, data residency, domain model, and the required authorization, quota, suppression, and reputation boundaries.

Read [provider selection](references/provider-selection.md) when selecting a service or evaluating current capabilities. MailChannels, cloud email services, and other managed platforms are candidates. Compare the isolation the selected plan actually provides. Subaccounts, tags, streams, and dedicated IPs solve different problems. Do not represent any of them as complete isolation without checking its boundaries.

Use [tenant lifecycle](references/tenant-lifecycle.md) for onboarding, data relationships, dispatch, events, suppression scope, abuse controls, and offboarding. Preserve these invariants:

- Derive the tenant from authenticated application context. Every sender, message, job, event, quota, and suppression operation includes and verifies that tenant scope.
- Send only through an active sender identity owned by the same tenant. DNS authentication does not alone prove that this customer owns the right to use a domain.
- Check tenant status, sender readiness, quota, preferences, and suppressions at dispatch. An enqueued job does not retain authority after a tenant or domain is disabled.
- Use separate provider credentials/accounts where required and available, with least privilege. Keep application authorization even when the provider offers isolation.
- Verify and durably ingest provider events, then resolve tenant ownership from trusted account/submission mappings. Caller-controlled metadata cannot determine ownership by itself.
- Scope recipient suppressions to their actual policy boundary. Do not let a tenant edit another tenant's suppressions, or bypass a platform safety restriction by moving to another provider.

Use an application message ID, durable outbox, bounded retries, and explicit handling of ambiguous acceptance. Check the provider's idempotency contract; SMTP submission does not imply exactly-once delivery. Provider delivery deferrals must not trigger duplicate application submissions. Preserve delivery/bounce/complaint history rather than allowing reordered events to overwrite it.

## Finish with observable isolation

Test with at least two tenants. Attempt cross-tenant sender use, forged metadata, foreign message/event IDs, simultaneous quota consumption, suppression edits, domain reclamation, stale queued jobs after suspension, and noisy-neighbor queue pressure. Test webhook authentication and duplicate/reordered events. Use sandbox/mocked sending unless real sends are authorized.

Deliver the application changes with a documented capability map: which boundaries the provider enforces, which the application enforces, remaining shared reputation/account limits, and how to suspend or offboard a tenant. Include domain onboarding instructions, event configuration, required secrets without values, metrics, and the material gaps found in the selected provider plan.
