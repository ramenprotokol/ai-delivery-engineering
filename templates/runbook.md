# Runbook: [Service Name]

> **Keep this current.** A runbook that describes last quarter's architecture is worse than no runbook — it wastes time during an incident. Update this file as part of every deploy that changes operational behavior.

---

## Metadata

| Field | Value |
|---|---|
| Service | e.g. Beacon notification delivery API |
| Version / tag | e.g. v2.14.1 |
| Runbook owner | Role (e.g. on-call engineer) |
| Last reviewed | YYYY-MM-DD |
| Next review due | YYYY-MM-DD |
| Severity contacts | See [Escalation](#escalation) |

---

## Service Overview

One paragraph. What does this service do, who depends on it, and what breaks if it goes down?

> **Example (synthetic):** Beacon is the transactional notification delivery API. It receives delivery requests from internal services (Orchard task management, Atlas CLI notifications), routes them to the appropriate channel (email, SMS, push), and returns a delivery receipt. Downstream services treat delivery receipts as authoritative — if Beacon is down, notifications stop and dependent workflows stall.

**SLO:** e.g. 99.9% of requests succeed within 500ms (p99), measured over a rolling 30-day window.

**On-call rotation:** [link to rotation schedule]

---

## Architecture and Dependencies

### System diagram

```text
[Caller service]
     |
     v
[Beacon API] ──> [Rate limiter (in-process)]
     |
     ├──> [Email provider: SendGrid]
     ├──> [SMS provider: Twilio]
     └──> [Push provider: FCM]
          |
          v
     [Delivery queue (Redis)]
          |
          v
     [Retry worker]
```

### Key dependencies

| Dependency | Type | Impact if unavailable | Timeout / retry policy |
|---|---|---|---|
| Email provider (SendGrid) | External API | Email delivery fails; requests queued | 10s timeout, 3 retries with exponential backoff |
| SMS provider (Twilio) | External API | SMS delivery fails; requests queued | 10s timeout, 3 retries |
| Delivery queue (Redis) | Internal | All delivery fails; no retry possible | Circuit breaker trips at 5 consecutive failures |
| Config service (Atlas) | Internal | Uses last known config; degraded but functional | 2s timeout; fail open |

### Ports and endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/deliver` | POST | Deliver a notification |
| `/status/{id}` | GET | Check delivery status by ID |
| `/health` | GET | Health check — returns 200 if healthy |
| `/metrics` | GET | Prometheus metrics scrape endpoint |

---

## Key Dashboards and Alerts

| Name | Type | Link | What to look at first |
|---|---|---|---|
| Beacon overview | Dashboard | [link] | Request rate, error rate, p99 latency |
| Delivery queue depth | Dashboard | [link] | Queue length; spikes indicate delivery backlog |
| `beacon_5xx_rate > 5%` | Alert | [link] | Fires if error rate exceeds 5% over 1 min |
| `beacon_queue_depth > 10000` | Alert | [link] | Fires if queue exceeds 10k messages |
| `beacon_p99_latency > 1000ms` | Alert | [link] | Latency regression |

---

## Common Operations

### Deploy a new version

```bash
# Verify the current version
atlas service status beacon

# Deploy a specific version
atlas deploy beacon --version v2.14.1

# Watch the deploy progress
atlas deploy logs beacon --follow

# Confirm healthy after deploy
atlas service health beacon
```

**Post-deploy checks:**

1. Open the Beacon overview dashboard — confirm error rate < 1% and latency is stable.
2. Run the smoke test: `atlas smoke-test beacon --env production`
3. Check the delivery queue depth — confirm it is not growing unexpectedly.

---

### Roll back to a previous version

```bash
# Roll back to the previous stable version
atlas rollback beacon

# Or roll back to a specific version
atlas rollback beacon --to v2.13.0

# Confirm rollback completed
atlas service status beacon
```

**When to roll back:** If error rate exceeds 5% within 10 minutes of a deploy and the cause is not immediately clear, roll back first — investigate after the service is healthy.

---

### Scale the service

```bash
# Check current replica count
atlas service replicas beacon

# Scale up
atlas scale beacon --replicas 6

# Scale back to baseline
atlas scale beacon --replicas 3
```

**Baseline:** 3 replicas. Scale up to 6 if queue depth exceeds 5,000 or request rate exceeds 1,000 RPS.

---

### Drain the delivery queue

Use only when the queue has stalled messages that need to be replayed.

```bash
# Check queue depth
atlas queue status beacon-delivery

# Replay stalled messages (dry run first)
atlas queue replay beacon-delivery --dry-run

# Execute replay
atlas queue replay beacon-delivery
```

---

### Rotate credentials

```bash
# Rotate the SendGrid API key
atlas secret rotate beacon/sendgrid-api-key

# Reload Beacon config without restart
atlas service reload beacon
```

---

## Troubleshooting

### Symptom → checks → action

| Symptom | First checks | Action |
|---|---|---|
| `beacon_5xx_rate` alert fires | 1. Check recent deploys. 2. Check dependency status (SendGrid, Twilio, Redis). 3. Check for unusual traffic spike. | If recent deploy: roll back. If dependency down: check circuit breaker status; enable fallback if available. |
| Queue depth growing, delivery not draining | 1. Check Redis is healthy. 2. Check retry worker logs. 3. Check provider API status pages. | If Redis is healthy and provider is up: restart retry worker (`atlas service restart beacon-retry`). If provider is down: queue will drain when provider recovers; no action needed unless depth exceeds 50k. |
| Latency spike (`p99 > 1000ms`) | 1. Check upstream provider latency. 2. Check Redis latency. 3. Check Beacon CPU and memory. | If provider latency is high: wait and monitor. If resource pressure: scale replicas up. If no external cause: check recent deploys. |
| Health check returning 503 | 1. Check all pods are running. 2. Check Redis connectivity from Beacon pod. 3. Check config service is reachable. | If pods are crashing: check pod logs (`atlas logs beacon --last 500`). If Redis is unreachable: escalate to infrastructure. |
| Specific tenant getting 429 errors | 1. Check rate-limit config for that tenant. 2. Check if `max_rps` is set correctly. | Update tenant rate-limit config: `atlas config set beacon tenants.<id>.max_rps <value>`. Reload config without restart. |

---

### Reading the logs

```bash
# Last 500 lines, all pods
atlas logs beacon --last 500

# Filter for errors only
atlas logs beacon --last 500 --level error

# Follow live
atlas logs beacon --follow

# Specific pod
atlas logs beacon --pod beacon-7f9d4c-xk2ql --last 200
```

**Key log fields:**

| Field | Meaning |
|---|---|
| `delivery_id` | Unique ID for this delivery request — use this to trace a specific message |
| `tenant_id` | Which tenant sent the request |
| `channel` | `email`, `sms`, or `push` |
| `provider_status` | HTTP status from the downstream provider |
| `retry_count` | How many times this message has been attempted |

---

## Escalation

Escalate when:

- The incident is SEV-1 (full outage) and is not resolved within 15 minutes.
- Root cause is not clear after 20 minutes of investigation.
- The fix requires a config or infrastructure change outside your access.
- Data loss is confirmed or suspected.

| Level | Who | When to engage |
|---|---|---|
| L1 | On-call engineer | First responder — always |
| L2 | Delivery engineer lead | After 15 min unresolved or if fix requires service change |
| L3 | Infrastructure / platform | If Redis, network, or provider infrastructure is involved |
| L4 | Engineering director | SEV-1 lasting > 30 min or confirmed data loss |

**Communication channel:** [link to incident Slack channel or bridge]

---

## Rollback

The rollback procedure for Beacon is covered in [Common Operations: Roll back to a previous version](#roll-back-to-a-previous-version) above.

**Rollback decision criteria:**

- Error rate > 5% within 10 min of a deploy → roll back, then investigate.
- Latency p99 > 2× baseline within 10 min of a deploy → roll back if cause is not immediately clear.
- Any confirmed data integrity issue → roll back immediately.

**Rollback is safe when:**

- No schema migrations were applied (check: `atlas migrations status beacon`).
- The previous version is known healthy (check: `atlas deploy history beacon`).

If a schema migration was applied, escalate before rolling back — a naive rollback may leave the database in an incompatible state.

---

## On-Call Notes

- Beacon's busiest period is 09:00–11:00 UTC weekdays (business notification volume spike). Alerts during this window are more likely to be real than false positives.
- The retry worker is a separate process (`beacon-retry`). It is not included in the main Beacon health check. Check it separately if queue depth is growing.
- Provider status pages: [SendGrid](https://status.sendgrid.com) — [Twilio](https://status.twilio.com)
- The delivery queue has a 24-hour message TTL. Messages older than 24 hours are dead-lettered, not replayed.
- If you change rate-limit config, `atlas service reload beacon` applies it without a restart. A full restart is not required.

---

## Related Documents

- [Architecture Decision Records](../examples/sample-architecture-decision-record.md)
- [Release plan template](../templates/release-plan.md)
- [Incident postmortem template](../templates/incident-postmortem.md)
- [Risk assessment template](../templates/risk-assessment.md)
- [Production readiness checklist](../checklists/production-readiness.md)

---

*Template version: see [CHANGELOG.md](../CHANGELOG.md) — [templates/](../templates/README.md)*
