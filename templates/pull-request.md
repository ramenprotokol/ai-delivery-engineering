# Pull Request Description Template

> This is the documentation version of the PR template. The `.github/PULL_REQUEST_TEMPLATE.md` file mirrors this for use directly in GitHub. Use this version as a reference, for offline review, or when writing PRs in tools that do not auto-populate the GitHub template.

---

## Summary

One to three sentences. What does this PR do and why? Written so a reviewer can understand the intent before reading the diff.

> **Example (synthetic):** Adds per-tenant rate limiting to the Beacon `/deliver` endpoint using SDK v3's `RateLimiter` class. This prevents a single high-volume tenant from crowding out others during traffic spikes. The change is behind a config flag and defaults to the existing behavior for tenants without explicit rate-limit settings.

---

## Type of Change

Mark all that apply.

- [ ] New feature
- [ ] Bug fix
- [ ] Refactor (behavior unchanged)
- [ ] Performance improvement
- [ ] Configuration change
- [ ] Documentation only
- [ ] Dependency upgrade
- [ ] Infrastructure / build change
- [ ] Breaking change — describe impact below

**If breaking change:** describe what breaks, who is affected, and what downstream teams need to do.

---

## What Changed and Why

Be specific. Link to the design doc, ADR, or issue that motivated this change. For each significant change, state both what changed and why that specific approach was chosen.

### Changed

- `beacon/middleware/rate_limit.py` — Added `TenantRateLimiter` wrapper class that reads per-tenant `max_rps` from config. Defaults to `None` (no limit) if the tenant has no explicit config entry, preserving existing behavior.
- `beacon/config/defaults.yaml` — Added `tenants` block with optional `max_rps` key per tenant. No existing values changed.
- `tests/integration/test_rate_limit.py` — Added 14 integration test cases covering: standard throttle, `max_rps = 0` edge case, missing tenant config (default behavior), and concurrent tenant isolation.

### Why this approach

The rate limiter is applied in middleware rather than at the handler level so it can be enforced consistently across all routes without per-handler changes. The config-file approach (rather than database-driven) was chosen because rate-limit values change infrequently and a file reload is faster to operationalize than a DB query on the hot path. See [ADR-007](../examples/sample-architecture-decision-record.md) for the full decision record.

---

## How It Was Validated

List every validation step. Do not just say "tests pass" — say what was tested, against what, and what the result was.

| Validation | Evidence |
|---|---|
| Unit tests | All existing tests pass; 14 new tests added and passing |
| Integration tests | Full integration suite passes including new rate-limit tests |
| `max_rps = 0` edge case | Explicit test added; verified it returns 429 rather than accepting all traffic |
| Load test | 5-min run at 500 RPS against staging; p99 latency 38ms before, 39ms after — within tolerance |
| Staging deploy | Deployed to staging YYYY-MM-DD; smoke test passed; queue depth stable |
| Backward compatibility | Tenants without a rate-limit config entry get `None` (no limit); verified by test `test_missing_tenant_defaults_to_no_limit` |

CI run: [link] — all checks green.

---

## AI Assistance

This section makes AI involvement explicit and traceable. See the [AI change log](../templates/ai-change-log.md) for the full entry.

**AI tools used in this PR:**

- [ ] Claude Code
- [ ] Cursor / Copilot
- [ ] Other: ___________
- [ ] No AI tools used

**What AI assisted with:**

| Task | AI tool |
|---|---|
| Drafted initial `TenantRateLimiter` class | Claude Code |
| Generated integration test scaffolding | Claude Code |
| Suggested `defaults.yaml` config schema | Claude Code |

**Human review and verification:**

- [x] Reviewed the full diff line by line
- [x] Verified AI-generated code against the intended design (not just "does it run")
- [x] Ran the test suite and confirmed all AI-generated tests actually test the right behavior
- [x] Caught and fixed at least one AI-generated issue: `None` tenant ID case was not handled — fixed by the engineer
- [x] Load test results reviewed by the delivery engineer (not just CI)
- [x] AI change log entry created and linked: [ACL-2025-001](../templates/ai-change-log.md)

**Statement:** AI accelerated this work. The delivery engineer reviewed all output, made architectural decisions, caught and fixed gaps, and is accountable for the result.

---

## Risk and Rollback

**Risk level:** Low / Medium / High

**Risk summary:** Describe what could go wrong and how likely it is. Be honest — this is read by the reviewer and the on-call engineer.

> **Example:** The `max_rps = 0` edge case is covered by a test, but the real risk is a misconfigured production config file. The post-deploy smoke test asserts `max_rps > 0` for all production tenants. If it fails, deploy is blocked before traffic promotion.

**Rollback:**

How to undo this change if it causes a problem in production. Be specific.

```bash
# Roll back to the previous version
atlas rollback beacon --to v2.13.0

# Confirm rollback completed
atlas service health beacon
```

For configuration-only changes: `atlas config revert beacon --to <previous-hash>`

**Is rollback safe?** Yes / No / Conditional — explain if conditional (e.g., if a schema migration is included, rollback needs coordination).

---

## Checklist Links

Before marking this PR ready for review, confirm:

- [ ] [Pre-merge checklist](../checklists/pre-merge.md) complete
- [ ] [AI code review checklist](../checklists/ai-code-review.md) complete (if AI was used)
- [ ] [Security review checklist](../checklists/security-review.md) complete (if applicable)
- [ ] [Testing checklist](../checklists/testing.md) complete
- [ ] Runbook updated if operational behavior changed: [runbook](../templates/runbook.md)
- [ ] Risk assessment updated if risks were identified: [risk register](../templates/risk-assessment.md)

---

## Screenshots / Output

Include terminal output, test results, dashboard screenshots, or before/after comparisons where they help the reviewer understand the change without running it themselves.

```text
# Example: smoke test output after staging deploy
$ atlas smoke-test beacon --env staging
  ✔ /health → 200 OK (12ms)
  ✔ /deliver → 201 Created (41ms)
  ✔ rate limit: tenant-A throttled at 100 RPS as configured
  ✔ rate limit: tenant-B unthrottled (no config entry → default behavior)
  ✔ max_rps=0 check: 0 tenants with invalid rate-limit config

All checks passed. (5/5)
```

---

## Reviewer Notes

Anything the reviewer should pay particular attention to, or questions you want feedback on.

> Example: I am not certain whether the config reload should be atomic or whether partial reload is safe. The current implementation reloads the whole config file at once, which I believe is safe, but I'd appreciate a second set of eyes on the `reload_config()` method in `rate_limit.py`.

---

*Template version: see [CHANGELOG.md](../CHANGELOG.md) — [templates/](../templates/README.md)*
