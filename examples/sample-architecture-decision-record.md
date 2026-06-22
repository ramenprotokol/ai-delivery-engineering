# ADR-0004: Retry Queue Backing Store for Beacon Delivery

**Status:** Accepted  
**Date:** 2026-04-11  
**Deciders:** Delivery engineer, backend engineer, on-call lead  
**Approver:** Engineering manager  
**Template:** [templates/architecture-decision-record.md](../templates/architecture-decision-record.md)

---

## Context

Beacon sends transactional notifications (email, SMS, webhook) on behalf of callers. When a downstream provider (e.g., an email gateway) returns a 5xx or times out, Beacon must retry delivery with exponential backoff — up to five attempts over roughly 32 minutes.

Before this decision, retries ran in-process: the worker that received the original request held the job in memory and re-attempted it. This worked at low volume. It broke under the following conditions:

- Worker restart (deploy, crash, OOM) silently dropped all pending retries.
- A spike of failures caused the in-process queue to grow unbounded, eventually triggering OOM kills.
- No visibility: there was no way to inspect queued retries, check delay times, or replay a job.
- Horizontal scaling added workers but did not share the retry backlog — each worker retried only its own jobs.

The team needed a durable, inspectable, operationally simple retry mechanism before enabling high-volume callers.

---

## Decision Drivers

| Driver | Weight | Notes |
|---|---|---|
| Durability across restarts | High | Lost retries = missed notifications; unacceptable for callers |
| Operational visibility | High | On-call must be able to inspect and replay jobs without a database console |
| Deployment simplicity | Medium | Avoid adding managed services the team does not already operate |
| Horizontal scaling | Medium | Multiple Beacon workers must share one queue |
| Cost | Low | Volume is modest; cost difference between options is negligible |
| Vendor lock-in | Low | The queue is an internal implementation detail; callers are not coupled to it |

---

## Options Considered

### Option A — In-process retry (status quo)

Retries are held in an in-memory structure inside the worker process. A background goroutine wakes up at each scheduled time and re-attempts delivery.

**Pros**

- Zero infrastructure dependencies.
- Simple to reason about in isolation.

**Cons**

- Jobs are lost on any worker restart (deploy, crash, OOM kill).
- In-memory growth is unbounded during failure storms.
- No shared state across workers — horizontal scaling does not help.
- No visibility into pending jobs or retry schedules.

**Verdict:** Does not meet the durability or scaling drivers. Rejected.

---

### Option B — Redis-backed queue (custom implementation)

Use Redis sorted sets: each pending retry is stored as a member with a score equal to its next-attempt Unix timestamp. A polling goroutine fetches jobs whose score is ≤ now, attempts delivery, and either removes the job (success) or re-inserts it with an updated score (failure, up to the attempt limit).

**Pros**

- Durable across worker restarts (Redis is already deployed for session caching).
- Shared across all workers; horizontal scaling works correctly.
- Sorted set structure gives natural visibility — `ZRANGE beacon:retries 0 -1 WITHSCORES` shows every pending job and its next-attempt time.
- No new infrastructure.
- Straightforward to implement (~200 lines, well-understood Redis primitives).

**Cons**

- Custom polling loop requires careful handling of clock skew and concurrent workers (must use `GETDEL`-style atomic operations or Lua scripts to avoid double-execution).
- Redis data is in-memory by default; requires AOF or RDB persistence enabled (already required by the existing session caching use case).
- On-call must understand Redis to operate the queue.

**Verdict:** Meets all drivers. Chosen.

---

### Option C — Managed queue (e.g., AWS SQS with delay queues)

Offload retry scheduling to a cloud-managed queue. SQS delay queues support per-message delivery delays up to 15 minutes; longer delays require re-enqueuing.

**Pros**

- Fully managed durability and scaling.
- No custom polling code.
- Dead-letter queues give visibility into failed jobs.

**Cons**

- 15-minute maximum delay requires re-enqueuing for the later retry attempts (attempts 4 and 5 in Beacon's backoff schedule exceed 15 minutes). Adds complexity.
- Adds a new managed service dependency; the team does not currently operate SQS.
- Message visibility timeout and at-least-once delivery require idempotency handling that the in-process model did not need.
- Higher operational surface area for the current team size.

**Verdict:** Operationally sound long-term. Not the right fit given current team size and the re-enqueue workaround required. Revisit if volume grows to warrant the tradeoff.

---

## Decision

**Option B — Redis-backed queue using sorted sets.**

Redis is already a production dependency. The sorted set primitive maps cleanly onto "scheduled retry at time T." Atomic pop-and-check via a Lua script prevents double-execution across concurrent workers. AOF persistence is already enabled. The queue is inspectable with standard Redis commands. No new services to procure, configure, or on-call for.

---

## Implementation Notes

Key names follow the pattern `beacon:retry:<environment>` (e.g., `beacon:retry:production`).

Each sorted set member is a JSON payload containing:

```json
{
  "job_id": "job_c8f3a1b2",
  "notification_id": "ntf_9d2e4f71",
  "channel": "webhook",
  "attempt": 3,
  "max_attempts": 5,
  "payload_ref": "s3://beacon-payloads/ntf_9d2e4f71",
  "next_attempt_at": 1712836200
}
```

Payloads are stored in object storage (not in Redis) to keep sorted set members small and Redis memory predictable.

The polling loop runs every 5 seconds. A Lua script atomically moves at most 50 members with score ≤ now into a worker-local processing list, preventing concurrent workers from claiming the same job.

Backoff schedule (jitter ±10%):

| Attempt | Nominal delay |
|---|---|
| 1 → 2 | 30 seconds |
| 2 → 3 | 2 minutes |
| 3 → 4 | 8 minutes |
| 4 → 5 | 22 minutes |

After five failures the job is moved to `beacon:retry:dead:<environment>` for manual inspection and replay.

---

## Consequences

**Positive**

- Retries survive worker restarts and deploys with no job loss.
- On-call can inspect the full retry queue with a single Redis command.
- Dead-letter set enables replay without re-triggering the original caller.
- No new infrastructure required.

**Negative / Risks**

- Redis becomes a harder dependency: if Redis is unavailable, Beacon cannot enqueue or process retries. Mitigated by the existing Redis HA setup (primary + replica, automated failover).
- Custom polling code introduces a maintenance surface. Mitigated by keeping the implementation small and covered by integration tests.
- Future volume growth may make a managed queue a better fit. This decision is revisable; the queue interface is abstracted behind a `RetryQueue` interface in the Beacon codebase.

---

## AI Assistance Note

The delivery engineer used Claude Code to draft the initial Lua script for the atomic pop operation and to enumerate edge cases (clock skew, partial failures, duplicate execution). The engineer reviewed the script against Redis documentation, tested it against a local Redis instance, and identified one missing `pcall` guard. The final script reflects human review and correction. Architecture ownership and the accept/reject decision are the engineer's.

---

## References

- [docs/validation-framework.md](../docs/validation-framework.md)
- [docs/risk-management.md](../docs/risk-management.md)
- [templates/architecture-decision-record.md](../templates/architecture-decision-record.md)
- [examples/README.md](README.md)
- Redis sorted set command reference: `ZADD`, `ZRANGEBYSCORE`, `ZREM`
- Internal Beacon retry interface: `internal/queue/retry_queue.go`
