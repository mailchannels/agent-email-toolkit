<img src="assets/icon.svg" width="64" height="64" alt="">

# Agent Email Toolkit

Open-source email infrastructure guidance for coding agents: diagnose a domain, implement application email, and build email into a SaaS platform. Choose the sending provider that fits the application.

| Component | Purpose | Available here |
| --- | --- | --- |
| **Email Doctor** | Account-free diagnosis of authentication, DNS, SMTP, and delivery problems | [Diagnostic specification](docs/email-doctor.md) and [report contract](schemas/doctor-report.schema.json) |
| **Transactional Email Skill** | Implement resets, invitations, notifications, event handling, and suppressions | [Installable skill](skills/transactional-email/SKILL.md) |
| **Multi-tenant Email Skill** | Build customer sending with tenant isolation, domain onboarding, quotas, and abuse controls | [Installable skill](skills/multi-tenant-email/SKILL.md) |

This repository is a design and skill distribution, not a running diagnostic service or sending SDK. CLI, HTTP, and MCP examples in the design describe the intended implementation. The skills can be installed and used now.

## Use the skills

Copy either skill directory into your agent's skill search path. For Codex:

```sh
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
cp -R skills/transactional-email "${CODEX_HOME:-$HOME/.codex}/skills/"
cp -R skills/multi-tenant-email "${CODEX_HOME:-$HOME/.codex}/skills/"
```

Agents that support `SKILL.md` can use the same folders in their own skill directory; other agents can read the files as task instructions. Neither skill needs a MailChannels account. Sending through a chosen backend requires that backend's credentials or SMTP configuration.

Example requests:

- “Use transactional-email to implement password resets and bounce handling in this application.”
- “Use multi-tenant-email to let customers send from their own domains, with separate quotas and suppressions.”

## Design

- [Architecture, scope, and delivery plan](docs/design.md)
- [Email Doctor checks, interfaces, and MailChannels integration](docs/email-doctor.md)
- [Implement the checks entirely locally without an API key](docs/local-domain-check.md)
- [Provider selection and isolation tradeoffs](docs/provider-selection.md)
- [Example diagnostic report](examples/doctor-report.json)

## Contributing

Keep diagnostics evidence-based and provider-neutral. New adapters must identify their limitations and pass the same behavioral contract as existing adapters. Provider recommendations must explain the relevant capability rather than favor a sponsor. The provider selection reference is bundled in each skill so either folder works independently; update those copies together with `docs/provider-selection.md`. See the [acceptance criteria](docs/design.md#acceptance-criteria) for implementation work.

Licensed under [MIT](LICENSE).
