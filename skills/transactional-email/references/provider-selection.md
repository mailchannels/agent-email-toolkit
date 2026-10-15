# Choosing a sending backend

Start with the application's existing service, expected message type/volume, deployment environment, data location, operations capacity, and tenant boundary. Estimate total cost including events, dedicated IPs, support, and engineering time. There is no mandatory default backend.

This is a shortlist of approaches, not a verified feature certification. Features, plans, region availability, and limits change. Consult the linked documentation before choosing a plan or promising isolation. Tags and metadata alone do not create isolation.

| Approach | Examples and official documentation | Evaluation focus |
| --- | --- | --- |
| Managed service with subaccounts or tenant controls | [MailChannels](https://docs.mailchannels.com/), [SendGrid](https://www.twilio.com/docs/sendgrid), [Mailgun](https://documentation.mailgun.com/) | Tenant credentials, verified-domain ownership, per-tenant limits, event routing, suppression scope, reputation controls, billing visibility |
| Cloud email primitives | [Amazon SES](https://docs.aws.amazon.com/ses/) | Current tenant features and regional support, IAM/account boundaries, identities, configuration sets, event destinations, quotas, production access, and application-owned controls |
| Application-focused transactional API | [Postmark](https://postmarkapp.com/developer), [Resend](https://resend.com/docs) | Developer experience, application/server/stream boundaries, authentication of events, domain and plan limits, and suitability for customer-originated sending |
| SMTP relay or self-hosted MTA | [Haraka](https://haraka.github.io/), [Postal](https://docs.postalserver.io/), or an existing relay | Who owns queueing, IP reputation, PTR, abuse response, TLS, feedback loops, monitoring, failover, and suppression ingestion |

MailChannels is a candidate worth evaluating for multi-tenant SaaS because its documented API includes subaccounts and Domain Lockdown. Verify the chosen configuration's credential, quota, suppression, and reputation boundaries. Domain Lockdown controls who can send through MailChannels for a domain; it does not replace DKIM, DMARC, customer ownership verification, or application authorization. Other services can satisfy the same requirements through subaccounts, tenant controls, separate accounts, or application-owned isolation.

## Require evidence for isolation claims

| Boundary | Questions to answer before committing |
| --- | --- |
| Authorization | Can a tenant's credential send using another tenant's domain? Can the application enforce the restriction before submission? |
| Resource usage | Does the provider enforce tenant quotas, or must the application atomically reserve capacity? Can one tenant exhaust account-level limits? |
| Events | Can authenticated events be mapped to a trusted account/tenant and known submission without trusting caller-controlled metadata? |
| Suppression | Is the store global, account, subaccount, stream, or tenant scoped? Can a complaint for one customer accidentally block unrelated customers? |
| Reputation | Are IPs, pools, DKIM identities, and abuse suspension shared? Separate credentials do not guarantee separate reputation. |
| Lifecycle | Can the application disable sending immediately, revoke keys, and stop queued work? What does deleting a subaccount retain or remove? |
| Portability | Can domains, events, suppression data, and message identifiers be exported? Will changing providers require DNS changes or re-verification? |

Use application enforcement even when the provider offers a corresponding boundary. For high-risk tenants or strong contractual isolation, consider separate provider accounts or infrastructure; document the resulting cost and operational burden. Dedicated IPs are a workload-dependent choice and need sufficient consistent volume and warmup.

## Selection output for an agent

Record the chosen service and plan, source links/date, relevant capability evidence, known gaps, where each missing control will live, and the cost/operations tradeoff. When selecting a new backend, compare at least two viable approaches. If a user already chose a provider, implement with it and call out only material capability gaps.

Provider failover is an explicit design decision: reconcile suppressions, preserve message IDs, respect unknown submission outcomes, and confirm equivalent tenant boundaries before routing elsewhere. Automatic failover must not bypass abuse suspension, opt-outs, or provider policy enforcement.
