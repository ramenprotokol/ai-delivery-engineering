# Release Readiness

A release is ready when it has been validated, not when it has been built. This document defines how go/no-go decisions are made, what gates a release must clear, and what happens after it ships.

---

## Definition of Done

A change is "done" when it meets all of the following — not just when the code compiles.

| Criterion | What it means |
|---|---|
| Acceptance tests pass | Every scenario in the test plan has a recorded result |
| No open P1/P2 bugs | All blocking issues are resolved or deferred with written justification |
| Documentation updated | CHANGELOG, runbook, README, and any ADRs are current |
| Security review complete | No high/critical findings open; medium findings triaged and tracked |
| Observability in place | Logs, metrics, and alerts exist before the release ships |
| Rollback path confirmed | Rollback tested or at minimum walk-through verified |
| Approver sign-off | At least one human reviewer who did not author the change has approved |

AI-generated code clears the same bar as hand-written code. The AI change log records what AI contributed; the approver confirms it was reviewed.

---

## Release Gates

Gates are evaluated in order. A failed gate blocks progression — it does not just produce a warning.

```text
┌────────────────────────────────────────────────────┐
│  Gate 1: Build & Unit Tests                        │
│  CI passes, no failing unit tests, no lint errors  │
└──────────────────────┬─────────────────────────────┘
                       │ PASS
┌──────────────────────▼─────────────────────────────┐
│  Gate 2: Integration Tests                         │
│  Service contracts hold, downstream calls verified │
└──────────────────────┬─────────────────────────────┘
                       │ PASS
┌──────────────────────▼─────────────────────────────┐
│  Gate 3: Security Review                           │
│  No high/critical open, secrets scan clean         │
└──────────────────────┬─────────────────────────────┘
                       │ PASS
┌──────────────────────▼─────────────────────────────┐
│  Gate 4: Observability Confirmed                   │
│  Dashboards updated, alert thresholds set          │
└──────────────────────┬─────────────────────────────┘
                       │ PASS
┌──────────────────────▼─────────────────────────────┐
│  Gate 5: Human Approver Sign-Off                   │
│  Non-author review complete, go/no-go recorded     │
└──────────────────────┬─────────────────────────────┘
                       │ GO
                  Production Deploy
```

The full checklist lives in [checklists/release-readiness.md](../checklists/release-readiness.md). Gates 1–4 can be verified by the delivery engineer. Gate 5 requires a second human.

---

## Go / No-Go Decision Table

| Condition | Decision | Action |
|---|---|---|
| All gates pass, approver confirmed | **GO** | Proceed to staged rollout |
| Gate 1 or 2 failing | **NO-GO** | Fix tests, re-run CI, restart gate sequence |
| Gate 3 has open high/critical finding | **NO-GO** | Resolve finding, re-run security checklist |
| Gate 4 incomplete (alerts not set) | **NO-GO** | Instrument, verify dashboards before deploying |
| Gate 5 — approver unavailable | **HOLD** | Wait; do not self-approve or skip |
| Gate 5 — approver explicitly rejects | **NO-GO** | Address feedback, re-submit for review |
| Known risk accepted with written justification | **GO with caveat** | Document risk, shorten canary window, brief on-call |

"GO with caveat" is not a workaround for skipping gates. It requires a written risk acceptance entry in the release plan before the deploy starts.

---

## Rollback and Reversibility

Every release plan must answer: "How do we undo this in under 10 minutes?"

### Reversibility tiers

| Tier | Description | Examples |
|---|---|---|
| **Fully reversible** | Redeploy previous artifact, no state change | Stateless API change, config update |
| **Mostly reversible** | Redeploy previous artifact, minor manual cleanup | New feature flag added but not toggled |
| **Partially reversible** | Rollback possible but requires data migration or coordination | Additive DB schema change |
| **Irreversible** | Cannot undo without data loss or customer impact | Destructive DB migration, external partner API change |

Irreversible changes require explicit sign-off from the approver on the irreversibility risk, a stricter canary window, and a confirmed customer communication plan before they ship.

### Rollback procedure (standard)

1. Confirm the previous artifact tag or commit SHA before deploying.
2. If deploy fails health checks within the canary window: trigger rollback immediately, do not attempt to patch forward.
3. Execute rollback: redeploy previous artifact, verify health check passes.
4. Page on-call if rollback takes more than 5 minutes or health check does not recover.
5. Open a postmortem ticket within 24 hours.

---

## Observability Requirements Before Release

A release does not go to production without these in place:

**Logs**

- Structured logs emitted for all critical paths (request received, processing start/end, error).
- Log level appropriate for production (info/warn/error; not debug by default).
- No sensitive data in log lines (PII, tokens, credentials).

**Metrics**

- Request rate, error rate, and latency (p50/p95/p99) tracked.
- Business-level metric tracked where applicable (e.g., notifications dispatched for Beacon).
- Dashboard updated to include new endpoints or services introduced by this release.

**Alerts**

- Error rate alert: fires if error rate exceeds threshold for 5 minutes.
- Latency alert: fires if p95 latency degrades beyond baseline.
- On-call rotation confirmed as active before release window opens.

If any of these are missing, Gate 4 is not cleared.

---

## Staged Rollout

All non-trivial releases use a staged rollout. "Non-trivial" means any change that touches business logic, authentication, data writes, or external integrations.

### Standard canary schedule

| Stage | Traffic | Soak time | Exit criteria |
|---|---|---|---|
| Canary | 5% | 30 minutes | Error rate stable, p95 latency within 10% of baseline |
| Partial | 25% | 1 hour | Same as canary, no escalating error patterns |
| Full | 100% | 24 hours (monitor) | No new incidents, metrics stable |

### Early exit conditions

Stop the rollout and roll back immediately if:

- Error rate rises more than 2x baseline at any stage.
- p95 latency increases more than 25% above baseline.
- Any on-call alert fires.
- A customer-visible bug is confirmed.

The delivery engineer is responsible for actively monitoring metrics during the canary window — not just setting alerts and walking away.

---

## Post-Release Verification

After reaching 100% traffic:

1. **Functional smoke test** — manually verify at least one end-to-end happy path within 30 minutes of full rollout.
2. **Metrics check** — confirm error rate and latency are stable after 1 hour at 100%.
3. **Log review** — scan logs for unexpected error patterns not caught by alerts.
4. **CHANGELOG published** — confirm the entry is accurate and accessible.
5. **Ticket closed** — mark the delivery ticket complete with a link to the deploy record.

Anything that fails post-release verification triggers the rollback procedure above, not a forward patch.

---

## Worked Example: Beacon v2.3 Release

**Change:** Beacon's `/dispatch` endpoint adds support for batched notification requests (up to 100 recipients per call). New request schema, new internal queuing path, new dead-letter queue (DLQ) for failed dispatches.

### Gate walkthrough

| Gate | Status | Notes |
|---|---|---|
| Build & unit tests | PASS | 247 unit tests passing, including 18 new tests for batch logic |
| Integration tests | PASS | Contract tests against downstream delivery service updated and passing |
| Security review | PASS | Input size cap (100 items) enforced; payload size limit set at 256 KB; no injection surface found |
| Observability confirmed | PASS | `beacon.dispatch.batch_size` histogram added; DLQ depth alert set at >50 messages; dashboard updated |
| Approver sign-off | PASS | Reviewed by on-call engineer; no blocking feedback; GO confirmed in PR |

**Reversibility tier:** Fully reversible. No schema migration. Previous artifact redeploys cleanly.

**Canary plan:** 5% for 30 min, then 25% for 1 hour, then full. Watch `beacon.dispatch.error_rate` and `beacon.dispatch.p95_latency`.

**Result:** All gates passed. Canary completed without incident. Promoted to 100% at 14:30. Post-release smoke test passed. CHANGELOG updated. Ticket closed.

---

## Related Documents

- [checklists/release-readiness.md](../checklists/release-readiness.md) — full checklist to work through before every release
- [templates/release-plan.md](../templates/release-plan.md) — release plan template
- [docs/validation-framework.md](validation-framework.md) — how changes are validated before they reach the release gate
- [docs/risk-management.md](risk-management.md) — risk identification and mitigation
- [templates/incident-postmortem.md](../templates/incident-postmortem.md) — postmortem template if the release causes an incident
