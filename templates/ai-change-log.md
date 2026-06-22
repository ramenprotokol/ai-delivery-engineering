# AI Change Log Template

This log is the auditability artifact for changes where AI tools contributed to the work. It answers a specific question: *what did AI do, what did a human verify, and who approved the result?*

Every entry covers one PR or one discrete change. The log does not replace a PR description or a test plan — it supplements them by making AI involvement explicit and traceable.

**Policy in one sentence:** AI accelerates the work. Humans own architecture, validation, decisions, and accountability.

---

## How to Use This Template

1. Copy this file to the relevant directory or maintain it as a running log per repository.
2. Fill in one entry per PR or change where AI tools were used.
3. The human reviewer section is mandatory — if no human verified the output, the change is not ready to merge.
4. Link the completed log entry from the associated PR description.

---

## Log Entries

---

### Entry: ACL-2025-001 (filled example)

*(Hint row — filled in with synthetic example to show the expected level of detail. Remove before using.)*

| Field | Value |
|---|---|
| Change ID | ACL-2025-001 |
| PR / Commit | `beacon/pull/214` — "Add per-tenant rate limiting to Beacon delivery pipeline" |
| Date | 2025-04-15 |
| Author (role) | Delivery engineer |
| Reviewer (role) | Senior delivery engineer |
| Approver (role) | Engineering lead |

#### What Changed

Added per-tenant rate limiting to the Beacon `/deliver` endpoint using the SDK v3 `RateLimiter` class. Changed three files: `beacon/middleware/rate_limit.py`, `beacon/config/defaults.yaml`, `tests/integration/test_rate_limit.py`.

#### Where AI Was Used

| Task | Files / Areas | AI Tool |
|---|---|---|
| Drafted initial implementation of `TenantRateLimiter` wrapper class | `beacon/middleware/rate_limit.py` | Claude Code |
| Generated integration test scaffolding for per-tenant throttle scenarios | `tests/integration/test_rate_limit.py` | Claude Code |
| Suggested config schema for `max_rps` per tenant in YAML | `beacon/config/defaults.yaml` | Claude Code |
| Explained SDK v3 changelog delta for `max_rps = 0` edge case | N/A (research) | Claude Code |

#### What AI Did Not Do

- Did not determine the rate-limit values for each tenant tier — those came from the delivery engineer after reviewing traffic baselines.
- Did not write the rollback procedure — that was written by the delivery engineer.
- Did not make architectural decisions about where rate limiting lives in the request pipeline — that was decided before AI was involved.

#### Human Review and Verification

| Review step | What the reviewer checked | Outcome |
|---|---|---|
| Code read | Reviewed the full diff line by line; confirmed AI output matched the intended design | Pass |
| Edge case test | Manually ran the integration suite including the `max_rps = 0` test case added by the engineer | Pass — all 14 tests pass |
| Config validation | Confirmed `defaults.yaml` schema is backward-compatible with existing tenant configs | Pass |
| Load test | Ran a 5-minute load test against staging at 500 RPS; confirmed p99 latency unchanged | Pass — 38ms p99 before, 39ms p99 after |
| AI output scrutiny | Checked that the AI-generated `TenantRateLimiter` wrapper handles the `None` tenant ID case (it did not initially — fixed by engineer) | Fixed and re-verified |

#### Validation Evidence

- CI run: all tests green — [link to CI run]
- Load test results: attached as `artifacts/load-test-2025-04-15.json` — [link]
- Integration test output: [link to test report]
- Staging deploy validated by: delivery engineer, YYYY-MM-DD

#### Residual Risks

| Risk | Severity | Mitigation |
|---|---|---|
| `max_rps = 0` in production config could reject all traffic | High | Post-deploy smoke test asserts `max_rps > 0`; see [risk register](../templates/risk-assessment.md) entry R-01 |
| Downstream consumers that cache rate-limit headers may behave unexpectedly | Low | Notified downstream teams; monitoring 429 rate for 48h post-deploy |

#### Approval

| Role | Decision | Date |
|---|---|---|
| Reviewer | Approved | 2025-04-15 |
| Approver | Approved | 2025-04-15 |

---

### Entry: ACL-YYYY-NNN (blank — copy this)

| Field | Value |
|---|---|
| Change ID | ACL-YYYY-NNN |
| PR / Commit | |
| Date | YYYY-MM-DD |
| Author (role) | |
| Reviewer (role) | |
| Approver (role) | |

#### What Changed

Describe the change in plain terms: what was added, removed, or modified, and which files were touched.

#### Where AI Was Used

| Task | Files / Areas | AI Tool |
|---|---|---|
| | | |
| | | |

#### What AI Did Not Do

List the decisions, designs, or verifications that were entirely human-driven. This is not optional — it makes the human contribution legible.

#### Human Review and Verification

| Review step | What the reviewer checked | Outcome |
|---|---|---|
| | | |
| | | |

#### Validation Evidence

List links to CI results, test reports, load test outputs, staging verification notes, or any other artifacts that prove the change was validated — not just that tests were run, but that a human looked at the results.

- CI run: [link]
- Test report: [link]
- Staging validation: [link or note]

#### Residual Risks

| Risk | Severity | Mitigation |
|---|---|---|
| | | |

#### Approval

| Role | Decision | Date |
|---|---|---|
| Reviewer | | |
| Approver | | |

---

## Quick Reference: AI Task Categories

Use consistent language in the "Where AI Was Used" column to make entries searchable and comparable over time.

| Category | Examples |
|---|---|
| **Drafting** | AI wrote the first version of a function, class, config, or doc |
| **Scaffolding** | AI generated boilerplate (test stubs, file structure, schema skeleton) |
| **Refactoring** | AI restructured existing code under human direction |
| **Research** | AI explained a library, API, or concept to inform a human decision |
| **Review assist** | AI flagged potential issues; human verified and decided |
| **Documentation** | AI drafted comments, READMEs, or runbook content |
| **Test generation** | AI produced test cases; human reviewed coverage and edge cases |

---

*Template version: see [CHANGELOG.md](../CHANGELOG.md) — [templates/](../templates/README.md)*
