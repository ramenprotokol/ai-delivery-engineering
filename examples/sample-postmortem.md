# Postmortem: Beacon Rate Limit Misconfiguration — Notification Drop (2026-04-19)

**Severity:** SEV-2  
**Duration:** 22 minutes (14:07–14:29 UTC)  
**Impact:** ~4% of outbound notifications rejected with HTTP 429 during the affected window  
**Status:** Resolved  
**Lead author role:** On-call engineer  
**Reviewers:** Delivery engineer, backend engineer, engineering manager  
**Template:** [templates/incident-postmortem.md](../templates/incident-postmortem.md)

> This postmortem is blameless. The goal is to understand what happened and prevent recurrence — not to assign fault to individuals.

---

## Summary

During the Stage 4 (100% rollout) of Beacon v1.3.0, a configuration error set the default caller quota to `5` requests per 60-second window instead of `500`. The error was introduced during a last-minute config edit to address a formatting concern flagged in the go/no-go review. The misconfigured value took effect when the rate limiter reloaded config at 14:07 UTC. For 22 minutes, roughly 4% of production notification requests were rejected with HTTP 429. The affected callers were those whose request rate exceeded 5 requests per minute — primarily webhook delivery and batch notification workloads. The issue was detected via alert at 14:09 UTC and resolved at 14:29 UTC by deploying the corrected config.

No data was lost. Callers with retry logic retried successfully after the fix. Callers without retry logic missed one delivery window; those notifications were replayed from the Beacon dead-letter set.

---

## Impact

| Dimension | Value |
|---|---|
| Duration | 22 minutes (14:07–14:29 UTC) |
| Notifications rejected (HTTP 429) | ~1,840 (est. 4% of traffic during window) |
| Notifications unrecoverable | 0 (all replayed from dead-letter set or retried by callers) |
| Callers affected | 3 of 47 production callers (those exceeding 5 req/min) |
| Customer-visible impact | Delayed notifications for 3 callers; no permanent data loss |
| Revenue impact | None identified |

---

## Timeline

All times UTC.

| Time | Event |
|---|---|
| **2026-04-19 12:30** | Engineering manager flags in go/no-go review: "double-check the default quota value — looks like a lot of zeros." |
| **13:45** | Delivery engineer edits `config/rate_limits.yaml` to address the comment. Intended change: confirm `500` is correct. Actual change: value is saved as `5` due to a digit dropped during editing. The edit is not reviewed by a second engineer before deploy. |
| **13:52** | Config change deployed to production via hot reload. No immediate error (the value `5` is syntactically valid). |
| **13:52–14:07** | Low-traffic period; no caller exceeds 5 req/min. Misconfiguration is not visible in metrics. |
| **14:07** | High-volume webhook batch job starts. Caller `beacon-webhook-batch` sends ~120 req/min. Rate limiter begins rejecting at request 6 of each 60-second window. |
| **14:09** | Alert fires: `BeaconRateLimiter429SpikeAlert` — 429 rate exceeds 1% over 5-minute window. |
| **14:10** | On-call engineer acknowledges alert. Initial hypothesis: a caller is exceeding their legitimate quota. |
| **14:14** | On-call engineer queries Redis: `ZCARD beacon:retry:production` shows 400+ pending retries — unusual spike. Begins suspecting a systemic issue rather than one misbehaving caller. |
| **14:16** | On-call engineer queries `beacon_ratelimiter_rejected_total` broken down by caller — sees 3 different callers being rejected, not just one. Escalates to delivery engineer. |
| **14:19** | Delivery engineer joins. Checks effective quota config via `/debug/config` endpoint: sees `default.max_requests: 5`. Immediately identifies the root cause. |
| **14:21** | Delivery engineer corrects `config/rate_limits.yaml`: `max_requests: 500`. Submits for review. Backend engineer approves in 2 minutes (expedited, both online). |
| **14:25** | Corrected config deployed. Rate limiter hot-reloads within 30 seconds. |
| **14:29** | 429 rate returns to 0. Alert resolves. |
| **14:35** | On-call engineer initiates replay of ~1,840 jobs from dead-letter set. |
| **15:10** | All dead-letter jobs replayed successfully. No notification loss confirmed. |
| **15:30** | Incident declared resolved. Postmortem scheduled. |

---

## Root Cause

A one-digit typo in `config/rate_limits.yaml` set `default.max_requests` to `5` instead of `500`. The error was introduced during a manual config edit made 26 minutes before the Stage 4 deploy. The edit was not peer-reviewed before deployment.

The contributing conditions that allowed a single-digit typo to reach production:

1. **No diff review before config hot-reload.** The release plan specified peer review for code changes but did not explicitly require a second pair of eyes on a "minor" config edit made post-approval.
2. **No schema validation with range checks.** The config loader validated that `max_requests` is an integer greater than zero. A value of `5` is syntactically valid; it is only semantically wrong given the expected traffic profile.
3. **Low-traffic window masked the error.** The 15-minute gap between deploy and first impact meant the misconfiguration was not caught during the initial "watch the metrics" period after deployment.
4. **Alert threshold (1% 429 rate, 5-minute window) was appropriate but not instantaneous.** By the time the alert fired at 14:09, the issue had been active for 2 minutes. This is acceptable — tightening the alert would increase noise. The delay is documented as expected behavior.

---

## What Went Well

- The `BeaconRateLimiter429SpikeAlert` fired within 2 minutes of the issue becoming visible in metrics — exactly as designed in the release plan.
- The on-call engineer correctly identified a systemic problem (multiple callers rejected) rather than chasing a single-caller explanation, which accelerated escalation.
- The `/debug/config` endpoint (added as part of v1.3.0 observability work) exposed the effective running config in under 30 seconds, cutting diagnosis time significantly.
- The rollback and correction path was fast: 10 minutes from root cause identified to fix deployed.
- The dead-letter queue introduced in ADR-0004 captured all rejected jobs, enabling full replay with no notification loss.
- No engineer felt pressure to skip the expedited approval step even under incident urgency — the backend engineer reviewed the corrected config diff before it was deployed.

---

## What Went Wrong

- A config change was made and deployed without a second review, despite the release plan requiring peer review for changes to production config.
- The config schema did not validate that `max_requests` is within a plausible range (e.g., ≥ 10 and ≤ 10000 for the default tier).
- The staging integration test suite did not include a test that loads the production config file and asserts the default quota is within a sane range.
- The low-traffic window between deploy and impact created a false sense that the config was correct.

---

## Action Items

| # | Action | Owner (role) | Due | Status |
|---|---|---|---|---|
| AI-01 | Add config review step to release checklist: any production config change made after go/no-go sign-off requires a second-engineer approval before deploy, regardless of perceived scope | Delivery engineer | 2026-04-26 | Open |
| AI-02 | Add range validation to config loader: `max_requests` must be ≥ 10 and ≤ 50,000; startup and hot-reload fail loudly if out of range | Backend engineer | 2026-04-30 | Open |
| AI-03 | Add a CI test that loads `config/rate_limits.yaml` and asserts `default.max_requests` is within the valid range | Delivery engineer | 2026-04-30 | Open |
| AI-04 | Add a staging smoke test that sends 20 requests from a test caller and asserts all 20 return HTTP 200 (would have caught a quota of 5 immediately) | Delivery engineer | 2026-04-30 | Open |
| AI-05 | Update release plan template to explicitly flag "config edits made post-approval are subject to the same review gate as code changes" | Delivery engineer | 2026-04-26 | Open |
| AI-06 | Document dead-letter replay procedure in the Beacon runbook so any on-call engineer can execute replay without escalation | On-call lead | 2026-05-05 | Open |

---

## Lessons Learned

**Config changes are code changes.** A one-character edit to a YAML file had the same blast radius as a code bug. "It's just a config tweak" is exactly the reasoning that bypasses controls. The release checklist should not distinguish between code and config when both affect production behavior.

**Schema validation catches syntax, not semantics.** A valid integer is not the same as a correct value. Range checks and sanity assertions in the config loader are cheap to write and would have caught this immediately at startup.

**Observability pays off quickly.** The `/debug/config` endpoint, the dead-letter queue, and the per-caller rejection metric were all added as part of this release. Each one directly shortened the time to diagnosis or recovery. Good observability is not optional on features in the request hot path.

**Low-traffic windows are not a deployment health check.** Seeing "no errors for 15 minutes" after a config deploy does not mean the config is correct — it may mean the traffic that would reveal the error has not arrived yet. Stage gate checks should include a synthetic traffic test, not just passive metric observation.

---

## AI Assistance Note

Claude Code drafted the initial timeline and contributing conditions list from notes the on-call engineer entered during the incident. The on-call engineer reviewed the draft, corrected the timeline at 14:14 (AI had placed the Redis query 3 minutes earlier than it actually occurred), added the "What Went Well" section (absent from the AI draft), and rewrote the lessons learned in their own words. Action items were authored entirely by the engineering team; the AI draft contained placeholders. The AI did not participate in the incident response itself.

---

## References

- [examples/sample-release-plan.md](sample-release-plan.md) — original release plan for Beacon v1.3.0
- [examples/sample-test-plan.md](sample-test-plan.md) — test plan for Beacon rate limiting
- [examples/sample-architecture-decision-record.md](sample-architecture-decision-record.md) — ADR-0004 (dead-letter queue)
- [docs/risk-management.md](../docs/risk-management.md)
- [docs/human-in-the-loop.md](../docs/human-in-the-loop.md)
- [checklists/release-readiness.md](../checklists/release-readiness.md)
- [templates/incident-postmortem.md](../templates/incident-postmortem.md)
