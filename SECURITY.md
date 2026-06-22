# Security Policy

## Overview

This is primarily a documentation and methodology repository. It contains no application code, no live services, no credentials, and no user data. The direct attack surface is low.

That said, security reports are taken seriously. Vulnerabilities in the repository itself (e.g., malicious CI workflow injection, exposed secrets, dependency issues in tooling) are in scope and will be addressed.

---

## Supported versions

| Version | Supported |
|---|---|
| 0.1.x | Yes |
| < 0.1 | No |

---

## How to report a vulnerability

**Use GitHub private security advisories.**

Open a confidential advisory at:
[https://github.com/ramenprotokol/ai-delivery-engineering/security/advisories/new](https://github.com/ramenprotokol/ai-delivery-engineering/security/advisories/new)

This channel is private between you and the maintainers. Do not open a public GitHub Issue for a security report — that exposes the vulnerability before it can be assessed.

Do not send reports to personal email addresses or social media channels.

---

## What to include in a report

A useful report includes:

- A clear description of the vulnerability
- Steps to reproduce (or a proof-of-concept if applicable)
- What you believe the impact to be
- Which version or commit you found it in

You do not need a working exploit to file a report. A credible concern is sufficient.

---

## Response window

Maintainers aim to acknowledge reports within **5 business days** and to provide an initial assessment within **10 business days**. Given the low severity surface of a documentation repository, most issues are expected to resolve quickly.

If you do not receive acknowledgment within 5 business days, you may follow up in the same advisory thread.

---

## Scope

### In scope

| Area | Examples |
|---|---|
| CI/CD pipeline | Malicious workflow injection, unsafe use of `pull_request_target`, secrets exposed in logs |
| Repository configuration | Public exposure of private data committed by mistake |
| Dependency vulnerabilities | npm packages used by CI tooling (markdownlint, link checkers) with known CVEs |
| Content injection | Maliciously crafted markdown or mermaid diagrams that exploit renderers |

### Out of scope

| Area | Reason |
|---|---|
| Spelling or grammar issues | Not a security concern — use a regular GitHub Issue |
| Style disagreements | Open a PR or Issue |
| Third-party services (GitHub itself, Actions runners) | Report those to GitHub directly |
| Theoretical vulnerabilities with no plausible exploit path | Out of scope for a documentation repo |

---

## Secure practices this repository follows

- **No secrets committed.** The `.gitignore` excludes `.env`, `.env.*`, `*.pem`, `*.key`, and similar files.
- **No credentials in examples.** All examples use synthetic placeholders (e.g., `Bearer <token>`, `API_KEY=<your-key-here>`).
- **Minimal dependencies.** CI tooling is limited to well-maintained, widely used packages.
- **Pinned Actions.** GitHub Actions workflows pin to major-version tags (e.g. `actions/checkout@v4`) rather than floating `@latest` references.
- **No user data.** The repository contains no personal data, customer records, or private system information. All examples use the synthetic Beacon/Atlas/Orchard universe.
- **Branch protection.** The `main` branch requires a pull request and at least one approval before merge. Direct pushes are disabled.

---

## Questions

For general questions about this repository, open a GitHub Issue at [ramenprotokol/ai-delivery-engineering](https://github.com/ramenprotokol/ai-delivery-engineering). Security reports go through the private advisory channel described above.
