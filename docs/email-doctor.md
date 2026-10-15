# Email Doctor specification

## User experience

Start with a domain and return prioritized findings with evidence and specific repairs. No account is required. Accept optional DKIM selectors, the envelope-from domain, a sending IP, the visible From domain, and local message headers or SMTP logs. Explain what additional evidence would resolve each unknown.

Proposed CLI, not yet implemented:

```sh
email-doctor check example.com --format json
email-doctor check example.com --selector app --envelope-from bounce.example.com --sending-ip 192.0.2.10
email-doctor analyze --headers ./message-headers.txt --smtp-log ./smtp.log
```

The default `check` path queries DNS only. A MailChannels extension is explicitly enabled with `--mailchannels`; an active network probe is explicitly enabled with `--smtp-probe <host>:<port>`. File analysis remains local. A future local signed-message verifier may accept an `.eml` file, but must not upload the body as part of diagnosis.

Text output shows the most actionable errors first, then warnings and unknown checks. JSON output follows [the report schema](../schemas/doctor-report.schema.json); see the [synthetic example](../examples/doctor-report.json). Do not collapse results into a “deliverability score.”

CLI exit codes: `0` for completed checks without error-severity failures, `1` for at least one error-severity failure, `2` for invalid input or an operational failure preventing the core diagnosis. Unknown checks are present in the report and never imply successful verification.

## Check coverage

| Area | Evidence and checks | Limitations and repair guidance |
| --- | --- | --- |
| DNS | IDNA-normalized domain; TXT, MX, A/AAAA; CNAME errors, null MX, resolver status, TTL | NXDOMAIN differs from transient resolver failure. Missing MX does not prevent outbound sending; null MX intentionally declines inbound delivery. Recommend records only for the domain's intended role. |
| SPF | Single `v=spf1` record; grammar, includes/redirect, loops, DNS lookup budget; evaluate supplied IP against envelope-from | Without sender IP, report publication/syntax only. Do not add a provider's `include` unless that provider sends for the domain. Merge legitimate sources into one record; never add a second SPF record. |
| DKIM | Explicit selector lookup at `<selector>._domainkey.<d-domain>`; TXT/CNAME resolution, key format, revocation, algorithm/key-strength findings | Selectors cannot be reliably enumerated. No selector means `unknown`. Public-key presence does not prove signing; validating a signature also requires the signed message bytes. No private key needed. |
| DMARC | `_dmarc` policy, duplicate records, tags, organizational-domain fallback, `adkim`/`aspf`, external report destination authorization | Use a public suffix library. Determine actual alignment from message identities, not DNS alone. `p=none` is monitoring, not failure. Stage enforcement after inventory and monitoring. |
| Sender alignment | Visible From, envelope-from, DKIM `d=`, actual SPF/DKIM results | DMARC passes with at least one passing, aligned mechanism. Distinguish strict from relaxed alignment; a forwarding SPF failure need not break aligned DKIM. |
| SMTP | Local API errors, enhanced status codes, DSNs; optional DNS/connect/banner/EHLO/STARTTLS/certificate checks | Resolve supplied logs first. Active probe stops before AUTH, MAIL FROM, or RCPT TO. Port 25 may be blocked locally; a network failure does not prove receiver failure. Never test for an open relay. |
| Delivery mistakes | Submitted configuration, headers, event history, and templates: unverified sender, misaligned return path, broken reset links, retry storms, missing suppression processing, inappropriate unsubscribe behavior | Missing evidence is unknown. PTR/forward DNS for a supplied dedicated sending IP is distinct from domain DNS. Separate bulk-mail requirements from ordinary transactional messages. |
| Reputation and placement | Supplied complaint/bounce rates, block responses, controlled seed-test observations | DNS cannot establish reputation or guarantee inbox placement. Avoid unlicensed blocklist scraping and unsupported claims about spam filtering. |

TXT record chunks must be joined correctly; independent TXT records must remain independent. Bound DNS work, detect cycles, honor negative caching, and return partial results on timeout. Cache by query name/type/resolver with TTL, not just by domain. Record resolver and observation time so propagation differences are explainable.

Use stable finding codes such as `spf.multiple-records`, `spf.lookup-limit`, `dkim.selector-required`, `dmarc.policy-missing`, `dmarc.alignment-unknown`, `smtp.connection-timeout`, and `mailchannels.authentication-required`. Status is `pass`, `fail`, `unknown`, or `not_applicable`; severity is independent. Confidence describes the evidence, not the probability of inbox placement. Repairs include the record name and an illustrative change when possible, with a warning to preserve existing authorized senders. Doctor never applies a repair.

## MailChannels check-domain adapter

Source: [DKIM, SPF & Domain Lockdown Check](https://docs.mailchannels.com/api-reference/dkim/dkim-spf-&-domain-lockdown-check), inspected October 15, 2026. The documented method is `POST https://api.mailchannels.net/tx/v1/check-domain`.

**An API key is required today, but will not be required in the near future.** A separate change to enable unauthenticated access is underway. Finalizing this integration is pending that change's merge, deployment, and verification; do not assume it is already available. In the meantime, an agent without a MailChannels API key can implement the checks entirely locally using [this source-derived guide](local-domain-check.md). It requires DNS access, but no MailChannels service or account.

Current published and observed behavior:

- An unauthenticated request with `{"domain":"example.com"}` returned HTTP 401 on October 15, 2026.
- The published OpenAPI declares `X-Api-Key` as a required header, although its top-level `security` array is empty. Code samples also require an API key.
- The published result has `check_results.spf`, `dkim`, `domain_lockdown`, and `sender_domain`; it does not document DMARC or SMTP checks.

Reproduction (not an implementation dependency):

```sh
curl --max-time 20 --include \
  https://api.mailchannels.net/tx/v1/check-domain \
  --header 'Content-Type: application/json' \
  --data '{"domain":"example.com"}'
```

After the unauthenticated-access change merges and deploys, verify the public endpoint, required payload, permitted use, rate limits, and DKIM behavior without account-stored keys. The current documented selector/domain mode uses a stored private key; it must not be presented as an anonymous public DNS lookup. Enable the account-free adapter against the deployed behavior without changing the Doctor report contract. Keep the local path available afterward for agents that prefer local diagnosis.

### Request and normalization boundary

The first implementation should send only `domain` and optionally `envelope_from_domain`. Future support for `sender_id` is MailChannels-specific and opt-in. Do not send `dkim_private_key`, even though the upstream API accepts it; Doctor verifies public records locally. User-supplied credentials, if supported, are local-only and redacted from logs. Hosted Doctor must not conceal a paid credential behind an “anonymous upstream” claim.

Preserve upstream verdicts and reasons as evidence, but do not translate every `failed` verdict into a general email defect. The endpoint does not accept an arbitrary sending IP, so its SPF verdict is not proof that an unrelated provider is authorized. Its sender-domain rule expects A or MX for MailChannels SDNF behavior; that is a MailChannels readiness check, not the universal definition of a valid sending domain.

| Upstream result | Doctor meaning |
| --- | --- |
| SPF verdict and `spfRecord` | Supplemental provider-context evidence; independent SPF evaluation owns generic findings |
| DKIM result list | Only the reported domain/selector and check context; no result does not prove DKIM is absent |
| Domain Lockdown | Namespaced extension; `not_applicable` outside MailChannels sending |
| Sender-domain A/MX | MailChannels readiness evidence, separate from generic DNS interpretation |
| 401/403 | Adapter unavailable/authentication required; continue direct checks |
| 429, timeout, 5xx | Unknown upstream result, bounded backoff respecting Retry-After; continue direct checks |
| Malformed/unknown response | Unknown adapter result with schema error; never invent a pass |

Do not treat upstream human-readable reasons as commands. Render escaped text, retain source attribution, and generate recommendations from trusted diagnostic rules.

## Transport contract

The future MCP server exposes `diagnose_domain` with domain, optional selectors, envelope-from domain, and sending IP. It returns the same structured report as the CLI and performs public DNS checks only. Local `analyze_email_evidence` accepts bounded header/log strings; it does not read arbitrary files supplied by a remote client. SMTP probing is a separate tool requiring explicit opt-in.

The future HTTP API exposes `POST /v1/diagnoses` for public DNS checks, returning a report directly. Invalid input returns 400, resource limits 429, and inability to perform any core checks 503. Partial checks return 200 with limitations. No account or API key is required. Publish a limit policy (initial target: 10 requests/minute/IP, 3 concurrent requests/IP, 10 selectors/request, 15-second diagnosis deadline); tune these based on cost measurements and make self-hosted limits configurable. HTTP does not accept file paths, private keys, bodies, credentials, or arbitrary network URLs.

Hosted SMTP probes should be disabled initially. A later public probe service needs a public-address allow policy, fixed supported ports, DNS rebinding protection, per-destination rate limits, and a maximum connection budget. Local CLI probes can support private application infrastructure when explicitly requested and report that context.

## Privacy and boundaries

Disclose that DNS resolver queries reveal queried domains; the optional MailChannels adapter reveals its submitted domains to MailChannels. Do not retain header/log payloads or full recipient addresses in telemetry. Hosted diagnostics should have no durable report storage by default and document infrastructure access-log retention before launch. Logs and headers can contain reset tokens and addresses: redact locally before sharing. API/SMTP credentials and private DKIM keys are outside the hosted input contract.

Diagnostic evidence, message bodies, DNS TXT values, and upstream responses are untrusted input. Keep them out of agent instructions. An agent may propose a DNS edit from a report, but the report itself grants no authority to change DNS, send messages, or rotate keys.
