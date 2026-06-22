# Incident Postmortem Template

> **Blameless by design.** This document exists to understand what happened and prevent recurrence — not to assign fault. Systems fail; the goal is to learn and improve them. Remove this notice before publishing.

---

## Metadata

| Field | Value |
|---|---|
| Incident ID | INC-YYYY-NNN |
| Severity | SEV-1 / SEV-2 / SEV-3 |
| Service / System | e.g. Beacon notification delivery |
| Date of incident | YYYY-MM-DD |
| Duration | e.g. 47 minutes |
| Date of postmortem | YYYY-MM-DD |
| Postmortem owner | Role only (e.g. on-call engineer) |
| Reviewers | Roles only |
| Status | Draft / In Review / Final |

---

## Summary

One to three sentences. What failed, what was affected, how it was resolved. Written so someone who was not involved can understand the incident without reading the rest of the document.

> **Example (synthetic):** On 2025-04-11, the Beacon API stopped delivering transactional notifications for approximately 47 minutes due to a misconfigured rate-limit header introduced in deploy `v2.14.1`. Approximately 12,000 notification requests were queued but not delivered. The fix was a one-line config rollback deployed at 14:23 UTC.

---

## Impact

Describe the blast radius in plain terms: who was affected, how many, and what they experienced.

| Dimension | Detail |
|---|---|
| Users / systems affected | e.g. All downstream consumers of Beacon `/deliver` endpoint |
| Scope | e.g. 100% of notification delivery for 47 min |
| Data loss | e.g. None — requests were queued and replayed |
| SLA / SLO breach | e.g. 99.9% availability SLO breached for the quarter |
| Customer-facing impact | e.g. Transactional emails delayed; no permanent failures |

---

## Timeline

All times in UTC. Include the moment the incident began (even if not yet detected), first detection, escalation, and resolution. Be specific.

| Time (UTC) | Event |
|---|---|
| HH:MM | First anomaly occurs (what changed — deploy, config push, external event) |
| HH:MM | First alert fires or first human notices |
| HH:MM | On-call acknowledges and begins investigation |
| HH:MM | Escalation to [role] — reason for escalation |
| HH:MM | Root cause identified |
| HH:MM | Mitigation applied (describe what) |
| HH:MM | Service confirmed healthy (describe signal used) |
| HH:MM | Incident declared resolved |
| HH:MM | Customer communication sent (if applicable) |

---

## Root Cause

State the root cause in one to two sentences. This is the technical condition that, if it had not been present, the incident would not have occurred. Do not conflate root cause with contributing factors or human error.

> **Example (synthetic):** The Beacon rate-limiter read its `max_rps` ceiling from an environment variable that was inadvertently set to `0` in the staging-to-production promotion script. A value of `0` was treated as "no limit" in the old SDK version but as "reject all" in v3.1.2, which had shipped two weeks earlier without a corresponding test for this edge case.

---

## Contributing Factors

List conditions that made the incident worse, harder to detect, or harder to resolve. These are not causes — they are amplifiers or gaps.

- The deploy pipeline did not run a smoke test against the rate-limit endpoint before promotion.
- The monitoring alert for `5xx` errors had a 5-minute evaluation window, which delayed detection by approximately 4 minutes.
- The runbook for this service had not been updated since the SDK upgrade.
- The on-call engineer was unfamiliar with the rate-limiter configuration because it is rarely changed.

---

## Detection

How was the incident first detected? Was it a machine alert or a human report? How long after the incident began was it detected?

| Question | Answer |
|---|---|
| How was it detected? | e.g. PagerDuty alert — `beacon_5xx_rate > 5%` |
| Who detected it? | e.g. on-call engineer |
| Time to detect (TTD) | e.g. 6 minutes after first anomaly |
| Was detection fast enough? | e.g. No — a 1-minute alert window would have caught it sooner |

---

## Resolution

Describe the steps taken to resolve the incident. Include what was tried but did not work, as well as what ultimately fixed it.

1. On-call engineer acknowledged alert and checked recent deploys.
2. Identified `v2.14.1` as the most recent deploy; began review of diff.
3. Found rate-limit env var change; tested hypothesis in staging — confirmed.
4. Initiated rollback to `v2.13.0` via the Atlas CLI: `atlas rollback beacon --to v2.13.0`.
5. Monitored delivery queue drain; confirmed healthy at 14:23 UTC.
6. Replayed queued notifications using Beacon's built-in retry mechanism.

**Time to mitigate (TTM):** e.g. 41 minutes after detection

---

## What Went Well

Be honest. Recognizing what worked reinforces good practices and helps calibrate what actually needs to change.

- The PagerDuty alert fired correctly and reached the right person.
- The rollback procedure was documented and the on-call engineer executed it without error.
- The notification queue preserved all in-flight requests; no permanent data loss.
- Communication to downstream teams was clear and timely.

---

## What Went Poorly

Be specific. Vague observations produce no action.

- The deploy pipeline did not have a post-deploy smoke test; this would have caught the issue in under 2 minutes.
- The alert evaluation window was 5 minutes, causing unnecessary delay.
- The runbook referenced a configuration path that no longer exists after the SDK upgrade.
- The env var change was in the deploy script, not in version control, so it was invisible in the PR diff.

---

## Action Items

Each action item must have a concrete description, an owner (by role), and a due date. Vague items ("improve monitoring") are not allowed here — make them specific.

| ID | Action | Owner | Due | Status |
|---|---|---|---|---|
| AI-01 | Add a post-deploy smoke test to the Beacon pipeline that validates `max_rps > 0` before traffic promotion | Delivery engineer | YYYY-MM-DD | Open |
| AI-02 | Reduce `beacon_5xx_rate` alert evaluation window from 5 min to 1 min | On-call / SRE | YYYY-MM-DD | Open |
| AI-03 | Update Beacon runbook to reflect SDK v3.x rate-limiter config paths | Delivery engineer | YYYY-MM-DD | Open |
| AI-04 | Move rate-limit env var out of the deploy script and into the versioned config file | Delivery engineer | YYYY-MM-DD | Open |
| AI-05 | Add a test case for `max_rps = 0` to the SDK integration test suite | Delivery engineer | YYYY-MM-DD | Open |

---

## Lessons

Distill the two to four things this incident teaches — things worth carrying into future work, not just fixes to this specific system.

1. **Behavioral changes in dependencies need explicit regression tests.** Upgrading the rate-limiter SDK without testing edge-case values meant a long-standing safe behavior silently changed.
2. **Configuration that lives outside version control is invisible in review.** If it cannot be diffed, it will be missed.
3. **Runbooks decay fast after infrastructure changes.** Treat runbook updates as part of every deploy that changes operational behavior.
4. **Detection lag compounds everything.** A 4-minute detection gap added more total impact than the resolution itself was long.

---

## References

- Incident alert: [link to PagerDuty / alert system]
- Deploy record: [link to deploy `v2.14.1`]
- Rollback record: [link to deploy `v2.13.0`]
- Related PR / diff: [link]
- Runbook: [link to runbook]
- Related action items / tickets: [links]

---

*Template version: see [CHANGELOG.md](../CHANGELOG.md) — [templates/](../templates/README.md)*
