# Release Plan: Beacon v1.3.0 — Rate Limiting

**Version:** Beacon v1.3.0  
**Feature:** Per-tenant rate limiting for outbound notification requests  
**Release type:** Feature release  
**Target date:** 2026-04-18  
**Release owner:** Delivery engineer  
**Template:** [templates/release-plan.md](../templates/release-plan.md)

---

## 1. Summary

Beacon v1.3.0 ships per-tenant rate limiting: a sliding window counter (backed by Redis) enforces configurable request quotas per tenant per 60-second window. Tenants who exceed their quota receive HTTP 429 with a `Retry-After` header. The feature was fully tested against the plan in [examples/sample-test-plan.md](sample-test-plan.md).

This release carries medium risk. Rate limiting sits in the hot path of every inbound request. A misconfiguration could reject legitimate traffic. The staged rollout and monitoring thresholds below are sized for that risk.

---

## 2. Prerequisites

| Item | Owner | Status |
|---|---|---|
| Test plan completed, all cases passing | Delivery engineer | Required before Stage 1 |
| Staging deploy stable for 24 h | Delivery engineer | Required before Stage 1 |
| Redis AOF persistence confirmed enabled in production | Backend engineer | Required before Stage 1 |
| Caller quota config reviewed and approved | Engineering manager | Required before Stage 1 |
| Runbook published ([templates/runbook.md](../templates/runbook.md)) | Delivery engineer | Required before Stage 1 |
| On-call engineer briefed | On-call lead | Required before Stage 1 |
| Rollback procedure tested in staging | Delivery engineer | Required before Stage 1 |

---

## 3. Rollout Stages

### Stage 1 — Internal traffic only (Day 1, 09:00 local)

Enable rate limiting only for internal Beacon test tenants (`tenant-internal-*`). Real production tenants are unaffected. Limits are set 10× above any observed internal request rate so no legitimate request is rejected.

**Duration:** 2 hours minimum  
**Gate to Stage 2:** Zero unexpected 429s in logs; `beacon_ratelimiter_redis_error_total` = 0; no latency increase on `beacon_request_duration_p99`.

---

### Stage 2 — 10% of production tenants (Day 1, 11:00 local)

Enable rate limiting for 10% of production tenants, selected by consistent hash on the tenant identifier. All selected tenants are pre-notified that limits will apply. Quota values: 500 req / 60 s per tenant (well above observed p99 tenant burst).

**Duration:** 4 hours minimum  
**Gate to Stage 3:** 429 rate < 0.1% of requests in this cohort; no tenant-reported incidents; `beacon_ratelimiter_redis_error_total` = 0; latency within 5% of pre-release baseline.

---

### Stage 3 — 50% of production tenants (Day 1, 15:00 local)

Expand to half of production tenants by the same hash ring. No quota changes from Stage 2.

**Duration:** 4 hours minimum  
**Gate to Stage 4:** Same thresholds as Stage 2. On-call engineer active.

---

### Stage 4 — 100% of production tenants (Day 2, 09:00 local)

Enable rate limiting for all production tenants. This is full rollout.

**Duration:** Hold for 24 hours before declaring the release stable.  
**Stable declaration:** Engineering manager sign-off after 24-hour observation window.

---

## 4. Configuration Reference

Rate limits are set in `config/rate_limits.yaml`. An excerpt for production:

```yaml
rate_limits:
  default:
    window_seconds: 60
    max_requests: 500
  overrides:
    tenant-highvolume-partner:
      window_seconds: 60
      max_requests: 2000
    tenant-internal-ops:
      window_seconds: 60
      max_requests: 10000
```

Config is loaded at startup and cached for 30 seconds. A hot reload does not require a process restart (SIGHUP triggers reload). Quota decreases take effect at the next window boundary to avoid retroactively rejecting in-flight requests.

---

## 5. Monitoring and Success Metrics

### Dashboards to watch

- Beacon request rate by status code (200 / 400 / 429 / 500)
- `beacon_ratelimiter_accepted_total` and `beacon_ratelimiter_rejected_total` (per tenant)
- `beacon_ratelimiter_redis_error_total` (should remain 0)
- `beacon_request_duration_p50`, `p95`, `p99` (latency must not degrade)
- Redis memory usage and hit rate

### Success metrics (24 h post full rollout)

| Metric | Target | Alert threshold |
|---|---|---|
| 429 rate (all tenants) | < 0.5% of requests | > 1% triggers rollback review |
| `beacon_ratelimiter_redis_error_total` | 0 | Any nonzero value triggers on-call page |
| Request latency p99 | Within 5% of pre-release baseline | > 10% increase triggers rollback review |
| Tenant-reported incidents | 0 | Any incident triggers rollback review |
| Retry queue depth (from ADR-0004) | No change from baseline | > 20% increase in queue depth triggers review |

### Alerts to configure before Stage 1

- `BeaconRateLimiter429SpikeAlert`: fires if 429 rate exceeds 1% of requests over any 5-minute window.
- `BeaconRateLimiterRedisError`: fires immediately on any Redis error from the rate limiter.
- `BeaconRequestLatencyDegradation`: fires if p99 exceeds 110% of the 7-day baseline.

---

## 6. Rollback Plan

Rollback is a config toggle — rate limiting can be disabled without a code deploy.

### Rollback trigger

Any of the following warrants immediate rollback discussion:

- 429 rate exceeds 1% across all tenants for more than 5 minutes.
- Any Redis error from the rate limiter in production.
- p99 latency exceeds 110% of baseline for more than 5 minutes.
- One or more tenants report unexpected request rejections.
- On-call engineer cannot determine root cause within 30 minutes of an anomaly.

### Rollback procedure

1. Set `rate_limits.enabled: false` in `config/rate_limits.yaml` and deploy the config change (no code change required; takes effect within 30 seconds via hot reload).
2. Confirm 429 rate drops to 0 in dashboards.
3. Confirm tenant-reported issues resolve.
4. Page the delivery engineer and engineering manager with a brief incident summary.
5. Open a postmortem issue (see [templates/incident-postmortem.md](../templates/incident-postmortem.md)).

**Rollback is reversible** — disabling the rate limiter returns Beacon to v1.2.x behavior exactly. No data migration is involved.

**Rollback rehearsed in staging:** Yes (2026-04-16). Toggle confirmed to take effect within 30 seconds.

---

## 7. Communication Plan

| Audience | Channel | Message | When |
|---|---|---|---|
| All production tenants | API changelog email | Rate limiting going live; quota values; 429 behavior; `Retry-After` header docs | 5 business days before Stage 2 |
| High-volume partners | Direct message from on-call lead | Same as above + offer to review quota needs | 7 business days before Stage 2 |
| Internal team | Team channel | Stage-by-stage rollout schedule; escalation path | Day of Stage 1 |
| On-call engineer | Runbook + briefing | What to watch, alert meanings, rollback steps | Before Stage 1 |

---

## 8. Go / No-Go Sign-Off

Sign-off is required before Stage 1 begins. Any "No-Go" blocks the release.

| Role | Decision | Notes | Date |
|---|---|---|---|
| Delivery engineer | Go / No-Go | — | — |
| Backend engineer | Go / No-Go | — | — |
| On-call lead | Go / No-Go | — | — |
| Engineering manager | Go / No-Go | — | — |

---

## 9. Post-Release Review

Scheduled 48 hours after Stage 4 completes. Agenda:

1. Review success metrics against targets.
2. Review any 429 incidents — were any unexpected?
3. Review Redis performance impact.
4. Identify quota values to revisit.
5. Confirm stable declaration or schedule follow-up.

---

## 10. AI Assistance Note

Claude Code drafted the initial rollout stage structure and the monitoring metric list from the feature spec. The delivery engineer reviewed the draft, adjusted stage percentages (initial draft proposed 25% / 50% / 100% — changed to 10% / 50% / 100% to reduce blast radius at first production exposure), wrote the rollback procedure based on direct experience with the staging rollback rehearsal, and added the communication plan. Rollback trigger thresholds are the engineer's judgment call, not AI output.

---

## References

- [examples/sample-test-plan.md](sample-test-plan.md) — test results for this release
- [examples/sample-architecture-decision-record.md](sample-architecture-decision-record.md) — ADR-0004 (Redis retry queue, same release)
- [docs/release-readiness.md](../docs/release-readiness.md)
- [docs/risk-management.md](../docs/risk-management.md)
- [checklists/release-readiness.md](../checklists/release-readiness.md)
- [templates/release-plan.md](../templates/release-plan.md)
- [templates/incident-postmortem.md](../templates/incident-postmortem.md)
