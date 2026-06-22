# Validation Framework

AI-generated code is fast. That speed is only useful if what ships is correct. This document defines how changes — regardless of who or what produced them — are validated before they are trusted.

The rule is simple: **a claim of completion is not evidence of completion.** Evidence is command output, test results, a passing CI run, a working endpoint, a diff that reads correctly. If it is not shown, it did not happen.

---

## Principle: Evidence Before Done

Every claim of "done" must be backed by observable evidence. This applies to human-written code, AI-generated code, and anything in between.

| Claim | Required evidence |
|---|---|
| "It compiles" | Build output with zero errors |
| "Tests pass" | Test runner output showing pass count and zero failures |
| "It works" | A real invocation — curl output, log line, screenshot |
| "No regressions" | Full test suite run, not just the changed unit |
| "It's safe" | Security checklist signed off, secrets scan clean |
| "Ready to release" | Release readiness checklist complete — see [release-readiness.md](release-readiness.md) |

Stating that something is done without showing the evidence is a gap in the delivery record. Reviewers must reject it.

---

## Validation Levels

Validation is layered. Each layer catches a different class of defect. Passing one layer does not substitute for another.

### Level 1 — Static Analysis

The change must not break the build. Linting must pass. Type errors must be zero.

```bash
# Example: Beacon API service
npm run lint
npm run typecheck
npm run build
```

What this catches: syntax errors, obvious type mismatches, import failures, style violations.

What this does NOT catch: incorrect logic, wrong behavior, integration failures.

### Level 2 — Automated Tests

The full test suite must pass. This includes unit tests, integration tests, and any contract tests.

```bash
npm test -- --coverage
```

Coverage is not the goal — meaningful coverage is. A function that routes, transforms, or validates data must have tests that cover the realistic paths through it.

What this catches: regressions, broken logic, incorrect transformations.

What this does NOT catch: runtime environment failures, external dependency behavior, performance characteristics.

See [testing-strategy.md](testing-strategy.md) for how tests are structured and what "meaningful coverage" means in practice.

### Level 3 — Acceptance Criteria

Every change originates from a requirement. Before a change is merged, each acceptance criterion in the ticket or design doc must be checked off explicitly.

For AI-generated implementations, this step is especially important. AI will implement what it inferred from the prompt, which is not always what was intended. The delivery engineer must read the acceptance criteria and verify the implementation matches — not assume alignment.

**Beacon example.** A ticket asks for rate limiting on `POST /v1/send` — max 500 requests per tenant per 60-second window, with a `429` response and a `Retry-After` header. After AI generates the middleware:

- [ ] Requests above 500 per 60-second window return `429`
- [ ] `Retry-After` header is present and reflects the correct reset window
- [ ] Requests below the limit pass through without added latency
- [ ] Rate limit state is per tenant, not global
- [ ] Limit is enforced over a rolling/sliding 60-second window (not reset on a fixed minute boundary)

Each item must be verified by running the actual behavior — not by reading the code.

### Level 4 — Runtime Behavior

For non-trivial changes, run the system and observe real output. Reading code is not the same as running code.

```bash
# Start Beacon locally
npm run dev

# Confirm the send endpoint is reachable
curl -s -w "\n%{http_code}" \
  -X POST http://localhost:3000/v1/send \
  -H "Authorization: Bearer test-key-001" \
  -H "Content-Type: application/json" \
  -d '{"channel":"email","to":"test@example.invalid","subject":"Validation test","body":"hello"}'
```

Expected: `200` with a delivery ID in the response body. Any other result requires investigation before the change proceeds.

### Level 5 — Security Review

Security review is not optional for changes that touch authentication, authorization, data ingestion, dependency versions, or configuration.

Required checks:

- Secrets scanner clean (no credentials, tokens, or keys committed)
- Input validation present on all external inputs
- No new packages with known CVEs
- Auth middleware still applied to protected routes
- No new environment variables exposed in client code

See [security-practices.md](security-practices.md) for the full security review process.

### Level 6 — Observability Check

A change that cannot be observed in production cannot be operated in production.

Before merging any feature or behavioral change:

- Confirm a structured log line is emitted for the new path
- Confirm the relevant metric (or a new one) is incremented
- Confirm an alert or monitor exists if the path can fail silently

**Beacon example.** A new delivery channel (SMS via Beacon's `POST /v1/send` with `channel: sms`) must emit a `delivery.attempted` event with `channel=sms`, a `delivery.succeeded` or `delivery.failed` event, and a duration metric. Without these, on-call has no visibility into whether the channel is working.

---

## Validation Matrix

This matrix maps change type to required validation levels. "Required" means the delivery engineer must produce evidence before the change is marked ready for review.

| Change type | L1 Static | L2 Tests | L3 Acceptance | L4 Runtime | L5 Security | L6 Observability |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| New feature | Required | Required | Required | Required | Required | Required |
| Bugfix | Required | Required | Required | Required | Conditional* | Conditional** |
| Refactor | Required | Required | Conditional*** | Required | Conditional* | No |
| Dependency bump | Required | Required | No | Conditional**** | Required | No |
| Config change | Required | No | Required | Required | Required | No |
| Documentation only | No | No | No | No | No | No |

**Conditional guidance:**

`*` Security review required if the bugfix or refactor touches auth, input handling, or permissions. Spot-check otherwise.

`**` Observability check required if the bug was a silent failure (no logs, no alerts fired when it should have). If it was already observed, confirm the fix resolves the signal.

`***` Acceptance criteria check for refactors: the behavior contract must be identical before and after. If the refactor changes no observable behavior, document that explicitly.

`****` Runtime check for dependency bumps: run a smoke test against the affected integration. For Beacon, if a transactional email dependency is bumped, send a test message and confirm delivery.

---

## AI-Specific Validation Rules

When AI tooling (Claude Code, Cursor, Copilot, or similar) generates or materially modifies code, two additional checks apply.

**Read the diff as a skeptic.** AI often generates plausible-looking code that is subtly wrong — correct syntax, wrong semantics. Read AI-generated diffs with more suspicion than human-written diffs, not less. Look specifically for: off-by-one errors in rate limits or pagination, missing error handling on async paths, hardcoded values that should be configurable, and tests that pass trivially because they assert nothing meaningful.

**Do not rely on AI to validate AI output.** If AI generated the implementation, a human must write or independently verify the tests. Asking AI to also write tests for its own output produces tests that pass because both the code and the tests share the same misunderstanding.

---

## What Reviewers Should Ask

When reviewing a PR that touches any non-trivial behavior:

1. Where is the test output? (Not the test file — the output of running it.)
2. Did you run this locally? What did you observe?
3. Which acceptance criteria did you check off, and how?
4. If AI generated this: did you read the diff line by line?
5. Can this fail silently in production? If yes, where is the alert?

These are not bureaucratic gates. They are the questions that catch the defects that reach production.

---

## Related

- [testing-strategy.md](testing-strategy.md) — Test pyramid, coverage philosophy, Beacon examples
- [release-readiness.md](release-readiness.md) — What must be true before a release
- [security-practices.md](security-practices.md) — Security review process
- [human-in-the-loop.md](human-in-the-loop.md) — Where human judgment is required, not optional
- [../checklists/ai-code-review.md](../checklists/ai-code-review.md) — Checklist for reviewing AI-generated changes
