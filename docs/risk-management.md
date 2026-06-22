# Risk Management

AI-assisted delivery introduces risk categories that don't appear in traditional software delivery — or that appear in new forms. This document names them, explains how they show up in practice, and maps each to a concrete mitigation in the methodology.

The goal is not to eliminate AI from the workflow. It is to stay aware of where AI is likely to be wrong and to verify before those errors reach production.

---

## Risk Taxonomy

### 1. Hallucinated or Incorrect APIs

**How it shows up:** An AI tool generates a call to a library function, SDK method, or API endpoint that does not exist, is deprecated, or behaves differently than the generated code implies. The code looks plausible and passes a linter. It fails at runtime.

**Likelihood:** High — especially for less-common libraries, recent API versions, or internal/proprietary SDKs the model was not trained on.

**Impact:** Medium to High — runtime failures, broken integrations, or silent bad behavior if the wrong method happens to exist and does something different.

**Mitigations:**

- Verify every AI-generated API call against official documentation or source code before merge.
- Run integration tests against real (or realistic stub) endpoints — not just unit tests mocking the same incorrect assumptions.
- Flag any library the AI is unfamiliar with in the AI change log; apply extra scrutiny to those calls.
- Never rely on AI to tell you whether its own API calls are correct. Check independently.

---

### 2. Silent Logic Errors

**How it shows up:** The generated code is syntactically correct and passes tests, but the business logic is subtly wrong — an off-by-one, a wrong comparison operator, an incorrect default, a missed edge case. Tests don't catch it because the tests were also AI-generated with the same flawed assumption.

**Likelihood:** Medium — increases when the AI writes both the implementation and the tests in a single generation pass.

**Impact:** High — these bugs reach production because they survive the test suite.

**Mitigations:**

- Write test cases independently from the implementation. Do not generate implementation and tests in one pass.
- Define acceptance criteria before generating code. Verify generated code against those criteria manually.
- Have a human read the business logic, not just the structure, during code review.
- Use property-based or boundary tests that the AI did not author.

---

### 3. Security Regressions

**How it shows up:** AI-generated code introduces an injection vulnerability, bypasses an authorization check, exposes an endpoint without authentication, uses an unsafe default (e.g., `CORS: *`), or inadvertently leaks information in error responses. These are rarely flagged by the AI itself.

**Likelihood:** Medium — AI models are trained to produce working code, not necessarily secure code.

**Impact:** Critical — security regressions can affect all users and may have regulatory or legal consequences.

**Mitigations:**

- Run every AI-generated change through the [security review checklist](../checklists/security-review.md) before merge.
- Do not trust AI to self-audit its output for security issues — ask explicitly and then verify independently.
- Treat any auth, authz, or input-handling code as high-risk regardless of how it was generated.
- See [docs/security-practices.md](security-practices.md) for the full secure-by-default approach.

---

### 4. Supply-Chain and Dependency Risk

**How it shows up:** An AI tool suggests adding a new dependency, upgrading a package, or switching to an alternative library. The suggested package is unmaintained, has known CVEs, was recently compromised, or introduces a transitive dependency conflict.

**Likelihood:** Low to Medium — AI tools don't always have current knowledge of CVE databases or package health.

**Impact:** High — a compromised or vulnerable dependency affects the entire system.

**Mitigations:**

- Audit every new dependency before adding it: check download volume, last publish date, open CVEs (Snyk, `npm audit`, `pip-audit`), and GitHub health signals.
- Prefer packages with long track records over newer alternatives suggested by AI.
- Pin dependency versions in production. Review diffs for unexpected transitive dependency changes.
- Run automated dependency scanning in CI.

---

### 5. Over-Trust of AI Output

**How it shows up:** The delivery engineer accepts AI-generated code, architecture suggestions, or analysis without independent verification — because the output is confident, well-formatted, and looks correct. Errors accumulate silently.

**Likelihood:** High — the fluency of AI output makes it easy to under-scrutinize.

**Impact:** Variable — can range from minor bugs to architectural mistakes that require significant rework.

**Mitigations:**

- Treat AI output as a draft from a competent but fallible junior engineer. Review it, do not rubber-stamp it.
- Maintain explicit human sign-off requirements — an approver who did not use AI for the review.
- Document AI contributions in the AI change log so reviewers know what to scrutinize.
- Build the habit of asking "what would this get wrong?" before accepting AI-generated logic.

---

### 6. Scope Drift

**How it shows up:** AI tools are responsive — they do what you ask. A vague or expanding prompt causes the AI to implement more than intended: refactoring unrelated code, changing interfaces that other services depend on, or introducing features that were not in scope. The diff is larger than expected.

**Likelihood:** Medium — increases with less specific prompts and iterative generation sessions.

**Impact:** Medium — unexpected changes increase review surface, can break downstream consumers, and obscure the actual change being reviewed.

**Mitigations:**

- Write specific, scoped prompts. Define what is in scope and explicitly state what is not.
- Review diffs fully — not just the files you expected to change.
- Reject AI-generated changes outside the agreed scope even if they look like improvements. Capture them as separate tickets.
- Use the AI change log to record what was prompted versus what was generated.

---

### 7. Data Handling and Privacy

**How it shows up:** AI-generated code logs more than it should (e.g., including request payloads containing PII), passes sensitive data to external services unnecessarily, or handles data in ways that violate retention policies. Prompts themselves may include sensitive data that is logged or sent to an AI provider.

**Likelihood:** Low to Medium — easy to miss in code review; AI tools may not know about data classification policies.

**Impact:** High — data exposure can have legal, regulatory, and reputational consequences.

**Mitigations:**

- Never include real personal data, customer data, or credentials in prompts. Use synthetic data.
- Review all logging statements in AI-generated code for PII exposure before merge.
- Confirm that any external calls made by new code do not include sensitive fields in request bodies or query parameters.
- Apply data classification labels to new fields and verify they are handled consistently.

---

## Severity / Likelihood Matrix

```text
             │  Low Impact  │  Medium Impact  │  High Impact  │  Critical Impact
─────────────┼──────────────┼─────────────────┼───────────────┼──────────────────
High         │              │  Silent Logic   │  Over-Trust   │  Hallucinated
Likelihood   │              │  Errors (some)  │  of AI        │  APIs
─────────────┼──────────────┼─────────────────┼───────────────┼──────────────────
Medium       │              │  Scope Drift    │               │  Security
Likelihood   │              │  Data Handling  │               │  Regressions
─────────────┼──────────────┼─────────────────┼───────────────┼──────────────────
Low          │              │                 │  Supply-Chain │
Likelihood   │              │                 │  Risk         │
─────────────┴──────────────┴─────────────────┴───────────────┴──────────────────
```

Cells marked are the primary position for each risk type. Individual instances may fall in adjacent cells depending on context.

---

## Risk Register Example

This is what a risk register entry looks like in practice. The full template is in [templates/risk-assessment.md](../templates/risk-assessment.md).

| ID | Risk | Category | Likelihood | Impact | Mitigation | Owner | Status |
|---|---|---|---|---|---|---|---|
| R-01 | Beacon's new batch dispatch logic contains off-by-one in recipient slice | Silent Logic Error | Medium | High | Independent test cases written against acceptance criteria; boundary tests at 0, 1, 99, 100 recipients | Delivery engineer | Mitigated |
| R-02 | AI suggested `lodash.merge` for config merging — may introduce prototype pollution | Supply-Chain / Logic | Low | High | Replaced with explicit `Object.assign` with typed inputs; no lodash dependency added | Delivery engineer | Resolved |
| R-03 | New `/batch` endpoint missing rate limiting | Security Regression | Medium | Critical | Rate limit middleware applied; verified in security checklist before merge | Delivery engineer + Approver | Resolved |
| R-04 | AI change log shows 60% of batch handler was AI-generated — reviewer may under-scrutinize | Over-Trust | Medium | Medium | Explicit note in PR: "high AI contribution — please read business logic, not just structure" | Approver | Active |

---

## How Risk Feeds Into Delivery Decisions

Risk is not just a tracking exercise. It affects real decisions:

- A high-impact open risk blocks the release gate (see [docs/release-readiness.md](release-readiness.md)).
- Unresolved critical risks require explicit written sign-off from the approver naming the risk and the acceptance rationale.
- Risk entries from this register feed the release plan (see [templates/release-plan.md](../templates/release-plan.md)) and postmortem (see [templates/incident-postmortem.md](../templates/incident-postmortem.md)) if a risk materializes.

---

## Related Documents

- [templates/risk-assessment.md](../templates/risk-assessment.md) — risk assessment template
- [checklists/security-review.md](../checklists/security-review.md) — security review checklist
- [docs/security-practices.md](security-practices.md) — secure-by-default practices
- [docs/human-in-the-loop.md](human-in-the-loop.md) — where human judgment is non-negotiable
- [docs/validation-framework.md](validation-framework.md) — how validation catches risks before production
- [docs/release-readiness.md](release-readiness.md) — how open risks affect go/no-go
