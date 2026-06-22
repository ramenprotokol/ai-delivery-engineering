# Release Plan: [Release Name or Version]

<!-- Title example: "Release Plan: Beacon v2.4.0 — Idempotent Retry Handling" -->

| Field | Value |
|---|---|
| **Status** | <!-- Draft / Approved / In Progress / Complete / Cancelled --> |
| **Release Date (target)** | <!-- YYYY-MM-DD --> |
| **Release Date (actual)** | <!-- Filled after release --> |
| **Release Owner** | <!-- Role of the person accountable for this release --> |
| **Approver (Go/No-go)** | <!-- Role of the person who makes the final call --> |
| **Linked Design Doc** | <!-- Relative path or URL --> |
| **Linked Test Plan** | <!-- Relative path or URL --> |
| **AI Assistance** | <!-- Example: "Claude Code drafted deployment steps from runbook; delivery engineer validated sequence and timing against actual system." --> |

---

## Release Summary

<!-- 2–4 sentences. What is shipping, why, and what risk category does this release carry (low / medium / high)? -->

## Scope — Changes in This Release

<!-- List every change included. Link to PRs, tickets, or ADRs where relevant. -->

| Change | Type | PR / Ticket | Risk |
|---|---|---|---|
| <!-- Description of change --> | Feature / Bugfix / Config / Dependency / Schema | <!-- Link --> | Low / Med / High |
| | | | |

**Changes explicitly excluded from this release:**

- <!-- Anything that was considered but is NOT shipping -->

---

## Pre-Release Checklist

Complete the [Release Readiness Checklist](../checklists/release-readiness.md) before proceeding to deployment. All items must be checked or explicitly waived with documented rationale.

- [ ] Release readiness checklist complete
- [ ] All tests passing in staging (see linked test plan)
- [ ] Database migrations validated in staging
- [ ] Feature flags configured correctly for target rollout percentage
- [ ] Monitoring dashboards reviewed — no existing alerts firing
- [ ] On-call engineer briefed and available during the release window
- [ ] Rollback tested or rollback path verified to be clean
- [ ] Communication sent to affected teams (if applicable)

---

## Deployment Steps

<!-- Step-by-step instructions. Precise enough to follow under pressure. Include the exact commands or links to runbooks where the full command lives. Do not abbreviate steps. -->

**Estimated duration:** <!-- e.g. 25 minutes -->

**Release window:** <!-- e.g. Tuesday 2026-06-23, 10:00–11:00 UTC -->

| Step | Action | Owner (role) | Verification |
|---|---|---|---|
| 1 | <!-- Confirm staging is green and all pre-release items are checked --> | Release owner | Checklist complete |
| 2 | <!-- Merge release branch / tag the commit --> | Release owner | `git tag` shows correct version |
| 3 | <!-- Trigger CI/CD pipeline --> | Release owner | Pipeline link shows green |
| 4 | <!-- Apply database migrations (if any) --> | Release owner | Migration log shows no errors |
| 5 | <!-- Deploy to production (canary / first region / first percentage) --> | Release owner | Deployment log shows success |
| 6 | <!-- Verify smoke tests pass against production --> | Release owner | See smoke test section below |
| 7 | <!-- Widen rollout or enable feature flag for full traffic --> | Release owner | See rollout strategy below |
| 8 | <!-- Confirm monitoring is clean for 15 minutes post-deploy --> | On-call | No new alerts; see success metrics below |
| 9 | <!-- Post release completion in team channel --> | Release owner | Message sent |

<!-- Add or remove steps as needed. -->

---

## Rollout Strategy

<!-- Describe how traffic is shifted to the new version. Choose the approach that matches the risk level of this release. -->

**Strategy:** <!-- Canary / Blue-Green / Feature flag / Staged percentage / Full immediate -->

| Stage | Traffic / Scope | Duration | Proceed Condition |
|---|---|---|---|
| Stage 1 | <!-- e.g. 5% of requests, or internal users only --> | <!-- e.g. 30 minutes --> | <!-- e.g. Error rate <0.1%, p99 latency <200ms --> |
| Stage 2 | <!-- e.g. 25% --> | <!-- e.g. 2 hours --> | <!-- Same metrics, plus manual spot check --> |
| Stage 3 | <!-- e.g. 100% --> | <!-- Ongoing --> | <!-- All metrics clean for 24 hours --> |

<!-- Compress to a single row for low-risk releases with no rollout gating. -->

---

## Smoke Tests

<!-- The minimum set of checks that confirm production is healthy immediately after deployment. These must be fast (< 5 minutes total) and directly observable. -->

| Check | How to verify | Pass condition |
|---|---|---|
| <!-- Health endpoint responds --> | `GET /health` | HTTP 200, `{"status":"ok"}` |
| <!-- Core happy-path works --> | <!-- Description of manual or automated check --> | <!-- Observable outcome --> |
| <!-- No spike in error rate --> | <!-- Dashboard link or metric name --> | <!-- Error rate ≤ baseline --> |
| <!-- Key downstream dependency reachable --> | <!-- How to check --> | <!-- Expected response --> |

---

## Monitoring & Success Metrics

<!-- What does "the release succeeded" look like in the data? Monitor these for the 24 hours following full rollout. -->

**Dashboards:** <!-- Link to monitoring dashboards -->

**Alerting:** <!-- What alerts are configured and at what thresholds -->

| Metric | Baseline | Alert Threshold | Success Threshold |
|---|---|---|---|
| <!-- Error rate --> | <!-- e.g. 0.05% --> | <!-- e.g. >0.5% for 5 min --> | <!-- e.g. ≤0.1% after 24h --> |
| <!-- p99 latency --> | <!-- e.g. 120ms --> | <!-- e.g. >500ms --> | <!-- e.g. ≤150ms --> |
| <!-- Throughput --> | <!-- e.g. 1,200 req/min --> | <!-- Drop >20% --> | <!-- Within 10% of baseline --> |
| <!-- Queue depth (if applicable) --> | <!-- e.g. 0 --> | <!-- e.g. >1,000 --> | <!-- e.g. ≤50 --> |

---

## Rollback Plan

<!-- What happens if the release needs to be reversed? This plan must be executable by on-call without escalation during an incident. -->

**Rollback trigger conditions:**

- Error rate exceeds <!-- threshold --> for more than <!-- duration -->
- P0 or P1 defect is discovered in production
- Go/no-go approver calls a halt

**Rollback steps:**

| Step | Action | Estimated Time |
|---|---|---|
| 1 | <!-- Disable feature flag / revert deploy / roll back image --> | <!-- e.g. 2 minutes --> |
| 2 | <!-- Verify rollback is active (smoke test) --> | <!-- e.g. 3 minutes --> |
| 3 | <!-- Confirm error rate returns to baseline --> | <!-- e.g. 5 minutes --> |
| 4 | <!-- Notify team and open incident ticket --> | <!-- e.g. 2 minutes --> |

**Schema migrations:** <!-- State whether migrations are reversible and how. "This migration is additive and requires no reversal." or "Down migration is scripted at migrations/20260621_rollback.sql." -->

**Estimated rollback time:** <!-- e.g. ~12 minutes end-to-end -->

---

## Communication

| Audience | Channel | Timing | Owner (role) |
|---|---|---|---|
| <!-- Engineering team --> | <!-- Slack channel or email list --> | <!-- Before release starts --> | <!-- Release owner --> |
| <!-- On-call / ops --> | <!-- PagerDuty / Slack --> | <!-- Day before --> | <!-- Release owner --> |
| <!-- Downstream teams (if API changes) --> | <!-- Email / docs update --> | <!-- 48h before --> | <!-- Release owner --> |
| <!-- Stakeholders (if visible change) --> | <!-- --> | <!-- --> | <!-- --> |

---

## Go / No-Go Sign-Off

**Go/no-go decision must be made no later than:** <!-- date and time, e.g. 2026-06-23 09:45 UTC -->

| Criteria | Status | Notes |
|---|---|---|
| Pre-release checklist complete | <!-- Yes / No --> | |
| All P0/P1 defects resolved | <!-- Yes / No --> | |
| Staging smoke tests passing | <!-- Yes / No --> | |
| Rollback plan verified | <!-- Yes / No --> | |
| On-call available | <!-- Yes / No --> | |

**Decision:**

| Approver (role) | Decision | Timestamp | Notes |
|---|---|---|---|
| <!-- e.g. delivery engineer lead --> | <!-- Go / No-go / Conditional go --> | <!-- YYYY-MM-DD HH:MM UTC --> | <!-- Any conditions or caveats --> |

---

## Post-Release Notes

<!-- Filled in after the release is complete. -->

**Actual release time:** <!-- YYYY-MM-DD HH:MM UTC -->

**Issues encountered:**

- <!-- Any unexpected events during the release -->

**Follow-up tickets opened:**

- <!-- Any work that was identified during the release window -->

**Lessons learned:**

- <!-- What would be done differently next time -->

---

*Related: [Release Readiness Checklist](../checklists/release-readiness.md) · [Test Plan template](test-plan.md) · [Incident Postmortem template](incident-postmortem.md) · [Release Readiness Docs](../docs/release-readiness.md)*
