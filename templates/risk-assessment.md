# Risk Assessment Template

Use this template before any significant change: a new feature, a large refactor, a dependency upgrade, an infrastructure change, or a release. The goal is to surface what could go wrong and ensure each risk has a named owner and a mitigation before work begins.

---

## Metadata

| Field | Value |
|---|---|
| Assessment ID | RISK-YYYY-NNN |
| Change / project | e.g. Beacon v3 rate-limiter upgrade |
| Author | Role (e.g. delivery engineer) |
| Reviewers | Roles |
| Date | YYYY-MM-DD |
| Status | Draft / Reviewed / Accepted |
| Linked ADR / Design Doc | e.g. [ADR-0004](../examples/sample-architecture-decision-record.md) |

---

## Context

One paragraph describing the change being assessed and why this risk assessment was initiated. Include scope, affected systems, and any known constraints.

> **Example (synthetic):** Beacon is upgrading its internal rate-limiter from SDK v2.x to v3.x to gain per-tenant throttling support. The change touches the delivery hot path and affects all downstream consumers of the `/deliver` endpoint. This assessment was initiated because the SDK changelog notes a behavioral change in how `max_rps = 0` is interpreted.

---

## Risk Register

Add one row per identified risk. A risk is something that *could* go wrong — not something that *has* gone wrong (that belongs in a postmortem).

| ID | Risk | Category | Likelihood | Impact | Severity | Mitigation | Owner | Status |
|---|---|---|---|---|---|---|---|---|
| R-01 | SDK interprets `max_rps = 0` as "reject all" instead of "no limit", breaking delivery for misconfigured tenants | Integration | Medium | High | High | Add explicit test for `max_rps = 0`; validate in staging before promotion | Delivery engineer | Open |
| R-02 | Post-deploy smoke test does not cover the rate-limit endpoint, allowing misconfiguration to reach production undetected | Process | Low | High | High | Extend smoke test suite to assert `max_rps > 0` on deploy | Delivery engineer | Open |
| R-03 | SDK upgrade introduces latency regression on the `/deliver` hot path | Performance | Low | Medium | Medium | Run load test against staging; compare p99 latency before/after | Delivery engineer | Open |
| R-04 | Rollback to v2.x is slow if v3.x schema changes are applied to config files | Operational | Low | High | High | Keep v2.x config as a versioned snapshot; test rollback path in staging | On-call / SRE | Open |
| R-05 | Downstream consumers that rely on undocumented SDK v2 behavior break silently | Dependency | Medium | Medium | Medium | Audit downstream callers; send advance notice of behavioral changes | Delivery engineer | Open |
| R-06 | | | | | | | | |

---

## Scoring Key

### Likelihood

| Score | Label | Meaning |
|---|---|---|
| Low | Unlikely | Has not happened before; requires multiple failures aligning |
| Medium | Possible | Has happened once or could plausibly occur with this change |
| High | Likely | Has happened before or is expected based on the nature of the change |

### Impact

| Score | Label | Meaning |
|---|---|---|
| Low | Minor | Degraded experience for a small number of users; recoverable quickly |
| Medium | Significant | Noticeable degradation for many users or a single customer-impacting failure |
| High | Severe | Full outage, data loss, SLA breach, or security event |

### Severity (Likelihood × Impact)

| | Low Impact | Medium Impact | High Impact |
|---|---|---|---|
| **Low Likelihood** | Low | Low | Medium |
| **Medium Likelihood** | Low | Medium | High |
| **High Likelihood** | Medium | High | Critical |

**Critical** risks require mitigation before the change proceeds. **High** risks require a documented mitigation and named owner. **Medium** and **Low** risks should be logged and monitored.

---

## Risk Acceptance

List any risks that are accepted without full mitigation, and state why.

| Risk ID | Rationale for acceptance | Accepted by (role) | Date |
|---|---|---|---|
| R-05 | Downstream audit is underway but will not complete before release; risk is monitored and consumers have been notified | Delivery engineer | YYYY-MM-DD |

---

## Pre-Change Checklist

Before the change is approved to proceed:

- [ ] All Critical and High risks have a mitigation and a named owner
- [ ] Mitigations that require code changes are tracked in the backlog or linked PRs
- [ ] Rollback path is documented and tested
- [ ] Risk register has been reviewed by at least one person other than the author
- [ ] This assessment is linked from the associated ADR, design doc, or release plan

---

## Review Notes

Space for reviewer comments. Each comment should include the reviewer's role and the date.

> *Role, YYYY-MM-DD:* Comment here.

---

*Template version: see [CHANGELOG.md](../CHANGELOG.md) — [templates/](../templates/README.md)*
