# Security Review Checklist

Use this checklist before merging any change that touches authentication, authorization, data handling, external inputs, dependencies, or AI-generated code. Every item must be checked — not skimmed. A passing CI pipeline is not a substitute for this review.

Related files: [pre-merge checklist](pre-merge.md) · [AI code review checklist](ai-code-review.md) · [security practices doc](../docs/security-practices.md) · [risk assessment template](../templates/risk-assessment.md)

---

## 1. Secrets and Credentials

- [ ] No API keys, tokens, passwords, or private keys are hardcoded in source files
- [ ] No secrets appear in commit history (check `git log -p` for the diff, not just the current state)
- [ ] `.env` files and secret stores are listed in `.gitignore` and absent from the PR diff
- [ ] CI/CD secrets are injected at runtime via the secrets manager — not via environment variable literals in workflow files
- [ ] AI prompts, prompt templates, and prompt logs do not embed secrets, tokens, or credentials in plain text
- [ ] Certificate files, private keys, and `.pem`/`.p12`/`.pfx` files are absent from the repository

## 2. Input Validation

- [ ] All external inputs (HTTP request bodies, query parameters, headers, path segments, file uploads) are validated against an explicit schema or allowlist before use
- [ ] Validation rejects unexpected field types, oversized payloads, and missing required fields — it does not silently drop or coerce them
- [ ] File upload handling enforces MIME type, extension allowlist, and maximum size server-side (not only client-side)
- [ ] Numeric inputs have explicit minimum/maximum bounds checks
- [ ] Inputs used in date/time parsing are validated against the expected format before parsing

## 3. Authentication and Authorization

- [ ] Every endpoint that returns or modifies data requires a valid authentication token or session — no unauthenticated read routes exist unless intentional and reviewed
- [ ] Authorization is enforced server-side on every request; client-supplied role or user ID claims are not trusted without verification
- [ ] Multi-tenant data: queries are scoped to the authenticated tenant/user — cross-tenant data leakage is not possible through ID enumeration
- [ ] Privileged operations (admin actions, bulk deletes, account changes) require elevated authorization and are logged
- [ ] Token expiry, rotation, and revocation are handled — stale tokens are rejected
- [ ] Password reset and account recovery flows do not leak whether an email/username exists in the system

## 4. Injection

- [ ] **SQL**: all database queries use parameterized statements or an ORM with binding — no string interpolation of user input into SQL
- [ ] **Command injection**: shell commands do not incorporate user input; if unavoidable, inputs are allowlisted and escaped
- [ ] **Template injection**: server-side template engines do not render user-supplied strings as template syntax
- [ ] **LDAP / NoSQL / XPath**: equivalent injection checks applied for any query language in use
- [ ] **Log injection**: user-supplied strings written to logs are sanitized to prevent newline injection that could spoof log entries

## 5. Cross-Site Scripting (XSS) and Content Security

- [ ] User-supplied strings are HTML-escaped before rendering in any web UI — no use of `innerHTML`, `dangerouslySetInnerHTML`, or equivalent with unescaped input
- [ ] Content-Security-Policy headers are set and do not use `unsafe-inline` or `unsafe-eval` without documented justification
- [ ] `X-Content-Type-Options: nosniff` and `X-Frame-Options` headers are present
- [ ] Open redirects: redirect targets are validated against an allowlist — user-supplied redirect URLs are not followed blindly

## 6. Dependency Vulnerabilities

- [ ] `npm audit`, `pip-audit`, `bundler-audit`, or the equivalent dependency scanner has been run and produces no high or critical findings
- [ ] Any new dependency added in this PR has been checked for known CVEs and is actively maintained (last release within 12 months)
- [ ] Lock files (`package-lock.json`, `Pipfile.lock`, `go.sum`, etc.) are committed and match the declared dependency versions
- [ ] AI-suggested packages were independently verified — the tool may hallucinate package names that resolve to typosquatting attacks

## 7. Least Privilege

- [ ] The service account, IAM role, or database user running this code has only the permissions it needs — no wildcard (`*`) policies unless justified and documented
- [ ] New infrastructure resources (S3 buckets, queues, databases) are private by default; public access requires explicit review sign-off
- [ ] New API keys or tokens were issued with the minimum scope required — not reusing a broader key for convenience
- [ ] Any elevated permissions granted temporarily during development have been revoked before merge

## 8. Sensitive Data Handling and Logging

- [ ] PII, payment data, health data, and other sensitive fields are not written to application logs in plain text
- [ ] Log statements that include request/response bodies redact sensitive fields before writing
- [ ] Sensitive data at rest is encrypted using an approved algorithm (AES-256 or equivalent); encryption keys are not stored alongside the data
- [ ] Retention policies are enforced — sensitive data is not stored longer than necessary
- [ ] API responses do not return fields the caller is not authorized to see (no "over-fetching" of sensitive fields)

## 9. Transport Security

- [ ] All external-facing endpoints enforce HTTPS — HTTP requests are redirected or rejected, not served
- [ ] TLS version is 1.2 minimum; TLS 1.0 and 1.1 are disabled
- [ ] Certificates are valid, issued by a trusted CA, and expiry is monitored with alerting before expiry
- [ ] Internal service-to-service calls use mutual TLS or equivalent authentication where the threat model warrants it
- [ ] Webhook endpoints verify the request signature before processing the payload

## 10. Error Handling and Information Leakage

- [ ] Error responses returned to clients contain a user-safe message and a correlation ID — not stack traces, file paths, SQL errors, or internal system names
- [ ] 404 and 403 responses are indistinguishable where enumeration is a concern (e.g., private resource IDs)
- [ ] Debug endpoints, admin routes, and health check details are not exposed without authentication in production
- [ ] Verbose error logging happens server-side only — clients receive the minimum needed to understand the error class

## 11. AI-Specific Checks

- [ ] AI-generated code has been read and understood by the reviewer — it was not merged solely because it passed tests
- [ ] Prompt inputs that include user-supplied content are validated and bounded in length before being sent to the model
- [ ] Model responses that are used in downstream logic (parsed, executed, or stored) are treated as untrusted input and validated
- [ ] No secrets, internal system names, or customer data appear in prompts logged for debugging or observability
- [ ] AI change log ([templates/ai-change-log.md](../templates/ai-change-log.md)) is updated if AI contributed substantially to this change

---

## Sign-off

| Role | Name / Handle | Date |
|---|---|---|
| Author | | |
| Security reviewer | | |

> A completed checklist does not guarantee no vulnerabilities exist. It documents that a structured review was performed. For changes with elevated risk (new auth flows, payment handling, admin capabilities), request a dedicated security review before merge.
