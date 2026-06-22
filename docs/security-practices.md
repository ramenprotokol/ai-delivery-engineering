# Security Practices

Security in AI-assisted delivery is not a separate phase. It is a set of habits applied continuously — when writing prompts, reviewing diffs, adding dependencies, and preparing to release. This document covers the practical defaults used in this methodology.

The goal is not theatrical security theater. It is honest, consistent practice that catches real problems before they ship.

---

## Secrets Handling

### The rule

No credential, token, API key, connection string, or secret of any kind belongs in a code repository — not in source files, not in comments, not in commit messages, not in test fixtures.

### How to handle secrets correctly

**Environment variables for runtime config**
All secrets are passed via environment variables. In local development, use a `.env` file (never committed). In deployed environments, use the platform's secret management (environment config, a secrets manager, or encrypted CI variables).

```bash
# .env (local only — in .gitignore)
BEACON_API_KEY=sk-dev-...
DATABASE_URL=postgres://localhost:5432/beacon_dev
```

**Worker environments (Cloudflare Workers pattern)**
Use `.dev.vars` for local secrets. Never commit `.dev.vars`.

```text
# .dev.vars (local only — in .gitignore)
BEACON_SIGNING_SECRET=whsec_...
```

**`.gitignore` — non-negotiable entries**

```gitignore
.env
.env.*
.dev.vars
*.pem
*.key
*.p12
*.p8
secrets/
```

**CI/CD secrets**
Secrets in CI pipelines are stored as encrypted environment variables in the CI provider — not hardcoded in workflow YAML files. Workflow files are code; treat them as such.

### AI prompts and secrets

Never paste credentials, tokens, connection strings, or real customer data into an AI prompt. AI conversations may be logged, cached, or used in ways outside your control.

If you need AI help debugging a secrets-related issue, redact the actual value:

```text
# Safe
DATABASE_URL=postgres://localhost:5432/mydb   ← use this in prompt

# Not safe
DATABASE_URL=postgres://admin:actualpassword@prod.db.example.com:5432/prod
```

---

## Dependency and Supply-Chain Hygiene

Every dependency is a trust decision. AI tools often suggest adding packages without knowing whether those packages are maintained, audited, or safe.

**Before adding any new dependency:**

1. Check the package's download count and publish date — abandonment is a warning sign.
2. Check for known CVEs: `npm audit`, `pip-audit`, Snyk, or the OSV database.
3. Confirm the GitHub repository is active and has a responsive maintainer.
4. Check the transitive dependency tree — a small package can bring in a large attack surface.
5. Prefer packages with long track records over AI-suggested alternatives you haven't heard of.

**Pin versions in production.** Floating version ranges (`^`, `~`, `*`) in production are a supply-chain risk. Pin to an exact version and update deliberately.

**Run dependency scanning in CI.** Automated scanning catches regressions between manual reviews.

---

## Reviewing AI-Generated Code for Security

AI models produce code that works. They don't consistently produce code that is secure. The following categories require explicit human review for every AI-generated change.

### Injection vulnerabilities

Check every place where user-supplied input reaches a query, shell command, template renderer, or external call.

```javascript
// AI might generate this — SQL injection if userId is not validated
const result = await db.query(`SELECT * FROM users WHERE id = ${userId}`);

// It should be this
const result = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
```

The fix is obvious in isolation. In a large generated diff, it is easy to miss.

### Authorization and access control

AI-generated route handlers often implement the happy path without verifying the caller has permission to perform the action.

Verify for every new endpoint or operation:

- Is the caller authenticated?
- Is the caller authorized to access this resource (not just any authenticated user)?
- Is authorization checked before the operation, not after?

### Unsafe defaults

Check for:

- `CORS: *` on APIs that should not be publicly accessible.
- `SSL: false` or certificate validation disabled in HTTP clients.
- `DEBUG: true` left enabled in generated configuration.
- Open firewall rules or permissive IAM policies in infrastructure-as-code.

### Leaked secrets in prompts

If you asked AI to help generate a config file, connection string, or credentials example, check the output for placeholder values that look real. Remove them before commit.

### Error response information disclosure

Check that error responses don't return stack traces, internal paths, database error messages, or config values to the caller.

```javascript
// AI might generate this — leaks internals
res.status(500).json({ error: err.message, stack: err.stack });

// Better
res.status(500).json({ error: 'Internal server error' });
// Log the full error server-side, not in the response
```

---

## Lightweight Threat Modeling Per Change

Before starting any non-trivial change, spend 10 minutes answering these questions. Write the answers in the design doc or PR description.

1. **What new attack surface does this change introduce?** New endpoints, new data inputs, new external calls, new user permissions.
2. **What is the worst thing an attacker could do with this change?** Think adversarially, even briefly.
3. **What data does this change touch?** Is any of it sensitive? Where does it flow?
4. **What happens if this component is compromised?** What's the blast radius?
5. **What are the trust boundaries?** Where does untrusted input enter the system?

This is not a formal threat model for every PR. It is a 10-minute habit that surfaces the issues worth thinking about before a reviewer sees the code.

For changes that involve authentication, authorization, payment flows, or data exports — do a more thorough review and document it explicitly.

---

## Least Privilege

Apply least privilege at every layer:

**Application level**

- Service accounts have only the permissions they need for their function — not admin by default.
- API keys are scoped to the minimum required operations.
- Database users have read-only access unless writes are required.

**Infrastructure level**

- IAM roles and policies grant minimum required access. No wildcard resource ARNs without justification.
- Network access is restricted to known sources. No `0.0.0.0/0` ingress rules without explicit sign-off.

**Developer access**

- Production access is not default. It is granted on a per-task, time-limited basis.
- No shared credentials. Each service and each developer has their own credential set.

**AI tools**

- AI tools that have filesystem or shell access operate with the minimum scope needed for the task.
- AI tools are not granted production credentials.

---

## Security Review Summary

Every change that touches auth, external integrations, data handling, or new endpoints goes through the security review checklist before merge.

The checklist lives in [checklists/security-review.md](../checklists/security-review.md). It covers:

- Secrets handling (no hardcoded values, `.gitignore` current)
- Dependency audit (new packages checked, no known CVEs)
- Input validation (all user input validated and sanitized)
- Auth/authz (every new endpoint checked)
- Error handling (no internal details leaked to callers)
- AI-specific review (injection surface, unsafe defaults, scope drift)

The security checklist is not optional for qualifying changes. It is a release gate.

---

## What This Does Not Cover

This document covers practices applied during development and delivery. It does not cover:

- Penetration testing or formal security audits — those are separate exercises.
- Incident response — see [templates/incident-postmortem.md](../templates/incident-postmortem.md).
- Compliance frameworks (SOC2, GDPR, HIPAA) — those require dedicated compliance programs beyond the scope of this methodology.

If a change requires compliance review, that is a separate gate and must be scoped into the delivery plan before work begins.

---

## Related Documents

- [checklists/security-review.md](../checklists/security-review.md) — security review checklist
- [docs/risk-management.md](risk-management.md) — security risk taxonomy and mitigations
- [docs/human-in-the-loop.md](human-in-the-loop.md) — where human judgment is required, including security decisions
- [docs/release-readiness.md](release-readiness.md) — security as a release gate
- [templates/risk-assessment.md](../templates/risk-assessment.md) — risk assessment template
