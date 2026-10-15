# Implement domain checks locally

MailChannels `check-domain` requires an API key today. A separate change is underway to remove that requirement in the near future. Until it merges and deploys, an agent can implement the following checks in the user's application or a standalone local script without a MailChannels API key. The local path remains useful afterward.

“Local” means the diagnostic logic and any supplied evidence stay on the user's machine, with DNS lookups through a configured resolver. It is not an offline check: live DNS still needs network access. Do not call `check-domain`, the internal SPF scanner, or the DKIM manager to implement this alternative. No diagnostic executable is included in this repository yet; this is an implementation recipe.

## Source and scope

The sequence below was traced in the `transactional-api` checkout at revision `9da5b827`, inspected October 15, 2026. Relevant source files, relative to that repository:

- `generated/go/check_domain.go`: input normalization, orchestration, identity selection, result aggregation.
- `client/dmarc/client.go`: delegates to an internal SPF scanner and maps SPF verdicts. Despite the package name, this path only consumes SPF results.
- `client/dns/client.go`: DNS lookups, Domain Lockdown parser, sender-ID wildcard matching, and address/MX checks.
- `client/dkimmanager/client.go`, `Verify`: delegates key verification to the DKIM manager and returns match/reason/status. The manager's internal verification algorithm is not implemented in these files.

The local equivalent below uses public DNS and standards libraries in place of internal services. It cannot discover account-stored DKIM keys, their lifecycle status, the authenticated MailChannels customer handle, or the service's configured egress IP automatically. Record those limitations rather than manufacturing matching results. DMARC and SMTP diagnosis are additional Doctor checks, not operations performed by this handler.

## 1. Normalize inputs and select identities

Accept:

- Required sending `domain`.
- Optional `envelope_from_domain`.
- Zero to ten DKIM settings with optional signing domain and selector; no private key is needed for public DNS checks.
- Optional sending IP for SPF evaluation.
- Optional expected MailChannels customer handle and sender ID, only if checking MailChannels Domain Lockdown. A customer handle is an identifier, not the API key.

The handler trims whitespace, lowercases the sending and envelope-from domains, and removes a trailing dot. It rejects invalid domain formats, invalid nonempty envelope-from domains, and more than ten DKIM settings. For a new local implementation, use a DNS/IDNA-aware validator with label/length checks rather than copying the handler's permissive regular expression.

Set `policy_domain = envelope_from_domain` when supplied; otherwise use `domain`. SPF and Domain Lockdown use `policy_domain`. Sender-domain address/MX checks use the original normalized `domain`. DKIM uses its explicit signing domain, defaulting to `domain`.

Deduplicate DKIM settings by normalized domain and trimmed selector, keeping the first occurrence. The source lowercases DKIM domains and trims selector whitespace. With no settings, the endpoint asks its key manager for the domain's stored keys; the local tool instead records `dkim.selector-required` as unknown and requests selectors from configuration or a DKIM-Signature header.

## 2. Evaluate SPF locally

The endpoint constructs a scanner request with:

| Scanner input | Handler value |
| --- | --- |
| Client IP | Configured MailChannels egress IP |
| HELO domain | `policy_domain` |
| Envelope sender | `postmaster@` + `policy_domain` |
| Recipient | `postmaster@mailchannels.com` |

The scanner request does not send an email. The recipient is scanner context, not permission to contact that address. The handler reads only the scanner's SPF result, policy record, and lookup error. It maps `pass`, `fail`, `softfail`, `temperror`, `permerror`, `neutral`, and `none` to its public verdict labels.

For local code:

1. Query TXT at `policy_domain`. Join chunks within each TXT record; do not join separate records. Identify SPF policies by their version. No policy is `none`; multiple policies produce `permerror`.
2. Use a maintained RFC 7208 implementation with a replaceable DNS resolver. Give it the actual sending IP, envelope sender, and HELO identity when known. Support mechanisms, qualifiers, recursive `include`, `redirect`, macros, IPv4/IPv6, and the RFC lookup/void-lookup limits. Detect cycles and preserve temporary versus permanent errors. Do not approximate SPF by searching for an `include` string.
3. If only the domain is available, check publication and syntax and report authorization as unknown. Do not guess a MailChannels egress IP or assume the developer machine's IP is the sender. A local check of another provider's IP answers a different, valid question from this endpoint's MailChannels-readiness check.
4. Return the raw SPF outcome and evidence. In Doctor, a timeout is unknown, not a policy failure; `softfail`, `neutral`, and `none` remain distinguishable in evidence and recommendations.

Changing DNS recommendations must preserve all legitimate senders. Never recommend a second SPF policy or a MailChannels include for a domain that does not use MailChannels.

## 3. Check Domain Lockdown only when relevant

The endpoint queries TXT at `_mailchannels.<policy_domain>` and compares the records with the authenticated customer handle and optional supplied sender ID. This check is MailChannels-specific. A local tool for another backend reports `not_applicable`; it does not mark the domain unhealthy for omitting this record.

To reproduce the inspected parser when MailChannels sending is relevant:

1. Fetch TXT records at exactly `_mailchannels.<policy_domain>`, with bounded retries for temporary DNS errors. There is no parent-domain search in this implementation.
2. The source recognizes records containing the exact case-sensitive string `v=mc1 `; a bare `v=mc1` is skipped. It then removes enclosing quotes, trims whitespace, lowercases the record and expected identifiers, and splits fields on spaces. Fields that do not split into exactly two `=`-separated parts are ignored.
3. Any recognized record can authorize the sender. An `auth=<handle>` field matching the expected customer handle passes. A `senderid=<id>` field matching a nonempty expected sender ID passes. A `sidw=<pattern>` field matching that sender ID also passes. These alternatives are OR conditions, not a requirement that every field match. The version field is skipped; unknown fields do not authorize anything.
4. For `sidw`, split pattern and sender ID into three parts at the first two `|` characters: account identifier, sender type, sender value. The first two parts match exactly after the source's lowercasing. The final part supports at most one unescaped `*`, matching any substring. A final part consisting only of `*` is invalid. `\\` encodes a literal backslash and `\*` a literal asterisk; other escapes, a trailing backslash, or multiple unescaped wildcards are invalid. With a wildcard, compare the unescaped prefix and suffix and require enough characters to avoid overlap; without one, compare the unescaped value exactly.
5. Return published records and whether any field matched the supplied identity. With neither expected identity available, report that publication can be inspected but authorization is unknown. Do not interpret “record exists” as “this account is allowed.”

These details describe the inspected implementation, not a proposed universal TXT parser. Recheck them against the eventual unauthenticated endpoint before claiming parity. A DNS failure is unknown in Doctor even though the source initializes this check to a failed verdict. Domain Lockdown does not prove customer domain ownership or replace SPF/DKIM/DMARC.

## 4. Check DKIM selectors using public DNS

The handler rejects an explicit signing domain outside the sending domain: it accepts equality or a suffix with a `.` boundary, so `sub.example.com` is allowed for `example.com` but `otherexample.com` is not. It then defaults an omitted signing domain, validates it, and requires a selector if a private key is supplied. This domain restriction is an endpoint constraint, not a general DKIM rule; a provider-neutral tool should inspect an explicitly supplied external signing domain and evaluate alignment separately.

The handler delegates `(customer handle, signing domain, selector, optional private key)` to the key manager. Omitted selectors/keys can select account-stored keys. It maps each returned match flag to `passed`/`failed` and includes reason and managed-key status. An empty manager result becomes failed/unknown. The client code does not show the manager's cryptographic implementation, so a local public-record check must not claim to duplicate it.

For a local alternative:

1. Get selectors and signing domains from application/provider configuration or the message's DKIM `s=` and `d=` values. Selectors cannot be reliably enumerated from a domain name.
2. Query `<selector>._domainkey.<signing-domain>` TXT, following CNAMEs with bounded depth and cycle detection. Preserve DNS errors and the distinction between absent records and lookup failure.
3. Parse the record with a DKIM-aware library. Validate tags, version where present, key type, base64 public-key encoding, and algorithm/key suitability. An empty `p=` revokes a key. Report malformed or conflicting records and use current algorithm/key-strength guidance rather than merely testing for nonempty text.
4. Return public-record readiness separately from signing verification. A public key in DNS does not prove that the application signs or has the matching private key. Provider-managed key lifecycle status is unavailable locally.
5. If the user has a raw signed message, verify its signature locally with a standards library and DNS. Full message bytes are needed for body-hash verification; copied headers alone are insufficient. If the user specifically needs a key-pair match and already has the private key locally, compare its derived public key with DNS using a cryptographic library without uploading or logging the secret. This optional local operation is outside Doctor's default public-DNS path.

No selector, no signed message, or no local key means those respective checks remain unknown. They do not establish that DKIM is absent or broken.

## 5. Check sender-domain addresses and MX

The endpoint checks both the address and MX lookup at `domain`, retains the individual results/reasons, and reports an overall pass if either succeeds. Its method is named `VerifyARecord`, but the implementation calls Go's `LookupHost`, which can return IPv4 and IPv6 addresses. A local reproduction must account for that rather than assuming only A is queried.

Query A, AAAA, and MX explicitly in local code so the report can explain which records exist. The endpoint's helper treats a nonempty MX result as a pass; Doctor should additionally recognize null MX as intentionally declining inbound mail, not an ordinary delivery route. Keep source-compatibility observations separate from general recommendations: the A-or-MX rule is MailChannels sender-domain readiness behavior, not proof of authentication or inbox placement, and missing MX alone does not prevent outbound sending.

## 6. Aggregate partial results

The handler continues after per-check service errors and returns the SPF, Domain Lockdown, DKIM, and sender-domain results. It adds help references when SPF or Lockdown is not passed or any DKIM result is not passed. Invalid top-level inputs instead produce HTTP 400.

Local implementations should retain this partial-result behavior while using Doctor's explicit `unknown` status for missing evidence and operational errors. Emit source, query, observation time, resolver, and actionable next steps using the [report contract](../schemas/doctor-report.schema.json). Preserve raw SPF outcomes and provider-specific scope rather than forcing everything into a single verdict.

Suggested orchestration (pseudocode, not a bundled executable):

```text
validate_and_normalize(inputs)
policy_domain = inputs.envelope_from_domain or inputs.domain
spf = local_spf_check(policy_domain, supplied_sender_context)
lockdown = local_lockdown_check(policy_domain, supplied_mc_identity)
dkim = local_public_dkim_checks(deduplicate(inputs.dkim_settings))
sender_dns = local_address_and_mx_checks(inputs.domain)
return doctor_report(spf, lockdown, dkim, sender_dns, limitations)
```

Add the [Doctor DMARC and SMTP checks](email-doctor.md#check-coverage) independently where evidence permits. Do not infer that either was checked simply because the upstream SPF client is named `dmarc`.

## Verification cases for generated code

Use deterministic resolver fixtures, not live domains alone. Cover envelope-from routing, duplicate SPF policies, SPF include cycles/limits, temporary DNS errors, absent selectors, revoked/malformed DKIM keys, CNAME loops, IPv6-only sender domains, null MX, and partial results. For the optional Lockdown parser, exercise each authorization field, wrong identities, standalone/multiple wildcards, escaped literal asterisks/backslashes, version case, and unavailable account context.

When the authentication change has merged and deployed, compare the new public endpoint against these fixtures where feasible, update its documented contract, and verify a no-key request succeeds before completing the MailChannels integration. Keep differences between local public-record checks and account-managed checks visible.
