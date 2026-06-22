# Bugfix Walkthrough: Duplicate Notifications Under Retry

This is a synthetic worked example of systematic bug investigation and resolution on Beacon (fictional transactional notification API). It demonstrates the investigation process described in [`docs/methodology-overview.md`](../docs/methodology-overview.md) — reproducing the problem, establishing the root cause, verifying the fix, and capturing a postmortem note.

---

## Bug summary

**System:** Beacon
**Symptom:** Some recipients received the same notification twice. Reports came from three separate tenants over a two-week period. The pattern: duplicates appeared only when the original send request had timed out on the caller's side and the caller retried.
**Severity:** High. Duplicate transactional notifications (payment confirmations, password resets) cause user confusion and support load.
**Tracking issue:** opened with reproduction steps before any fix was attempted.

---

## Stage 1: Reproduce

Before touching code, the delivery engineer needed a reliable reproduction. "Works on the caller's side when they retry" is not a reproduction — it is a symptom description.

**Questions to answer first:**

- Is the duplicate always the result of two worker executions, or could it be the delivery provider sending twice?
- Does it happen for all notification types or specific ones?
- Is there a timing dependency (race condition, not just retry)?

**Reproduction attempt 1:** Send a request to Beacon, kill the client before the response arrives, retry with the same payload. Result: the notification arrived once. No duplicate. This ruled out a naive "send twice on 5xx" theory.

**Reproduction attempt 2:** Send a request to Beacon. Introduce a 30-second delay in the worker (simulated by temporarily increasing processing time in a test environment). The caller's HTTP client times out at 10 seconds (default). The caller retries. Result: **two notifications delivered.**

Reliable reproduction established. The delivery engineer wrote the reproduction steps into the tracking issue before continuing.

---

## Stage 2: Root cause investigation

### Initial hypothesis

The caller's retry sends a second HTTP request. If the first request's worker job had already been enqueued before the HTTP response timed out, Beacon would process both the original job and the retry job independently — two sends.

The delivery engineer asked AI to help map the request path:

**Prompt given to AI:**

```text
Here is Beacon's send endpoint handler (pasted). Here is the job queue enqueue call.
Here is the worker's consume loop. Trace what happens when:
1. A POST /v1/send request arrives.
2. The worker picks up the job but takes longer than the caller's timeout.
3. The caller retries with identical payload.
Describe the execution path step by step. Do not suggest fixes yet.
```

AI traced the path and identified that:

- The request handler enqueues a job and responds `202 Accepted`.
- The worker processes the job asynchronously.
- If the handler's response never reaches the caller (timeout), the job is already enqueued.
- The retry enqueues a second job with a new job ID.
- The system had no mechanism to detect that the payload was a retry of an already-enqueued or already-processed request.

The delivery engineer verified this trace by reading the handler code and the worker consume loop directly. AI's description was accurate.

### Deeper question: where should idempotency be enforced?

Beacon already accepted an `Idempotency-Key` header — it was documented. The delivery engineer checked the handler: the idempotency key was validated at the HTTP layer to prevent duplicate HTTP requests from being accepted, but validation only checked an in-memory map with a 5-second TTL.

The bug was now precise:

- The 5-second in-memory TTL was shorter than realistic caller timeout windows (10–30s).
- After TTL expiry, a retry with the same idempotency key was treated as a new request.
- The first job was still being processed in the worker when the second job arrived.

**Root cause:** Idempotency key TTL (5 seconds, in-memory) was too short relative to realistic caller timeouts, and idempotency state was not persisted past the HTTP response — the worker had no access to it.

The delivery engineer documented this in the tracking issue with a timeline diagram before writing any fix.

### What AI got right and what the human verified

AI correctly traced the execution path from the handler to the worker. It did not identify the TTL mismatch — that required reading the idempotency key implementation, which the delivery engineer did independently. The root cause was confirmed by the human, not inferred from AI output.

---

## Stage 3: Fix

### Design decision

Two options were considered:

**Option A:** Extend the in-memory TTL to 60 seconds.

- Fast to implement.
- Does not survive a process restart or horizontal scale-out.
- Rejected: Beacon runs multiple instances; in-memory state is not shared.

**Option B:** Move idempotency key storage to Redis, with a TTL of 24 hours.

- Redis is already deployed (used by rate limiting — see [`examples/feature-delivery-walkthrough.md`](feature-delivery-walkthrough.md)).
- Key format: `idempotency:<tenant_id>:<idempotency_key>` → value: `{status: "processing"|"complete", result: ...}`.
- If the key exists and status is `processing`: return `202` (the first request is still in flight).
- If the key exists and status is `complete`: return the cached result immediately (no re-enqueue).
- If the key does not exist: proceed normally, set key to `processing`.
- Chosen.

**What AI assisted with:**

The delivery engineer described the Redis key design and the state machine (`processing` → `complete`) and asked AI to implement the storage functions:

```text
Implement two functions using the existing Redis client:
1. checkIdempotencyKey(tenantId, key): returns null | {status, result}
2. setIdempotencyKey(tenantId, key, status, result?, ttlSeconds?): void
Key format: idempotency:<tenantId>:<key>
Default TTL: 86400 seconds (24h)
Serialize result as JSON.
```

AI produced working implementations. The delivery engineer reviewed them:

- TTL was applied correctly.
- JSON serialization handled null result correctly (for the `processing` state, result is null).
- No issues found. Merged as-is after review.

The handler integration (checking the key before enqueuing, updating status after worker completion) was written by the delivery engineer.

---

## Stage 4: Regression test

The fix introduced a clear state machine. The delivery engineer wrote tests for each transition:

```typescript
describe('idempotency key behavior', () => {
  it('enqueues and returns 202 on first request', async () => {
    const res = await sendRequest({ idempotencyKey: 'key-001' });
    expect(res.status).toBe(202);
    const state = await redis.get('idempotency:tenant-abc:key-001');
    expect(JSON.parse(state).status).toBe('processing');
  });

  it('returns 202 without re-enqueuing on retry during processing', async () => {
    await redis.set(
      'idempotency:tenant-abc:key-001',
      JSON.stringify({ status: 'processing', result: null }),
      'EX', 86400
    );
    const enqueueSpy = jest.spyOn(jobQueue, 'enqueue');
    const res = await sendRequest({ idempotencyKey: 'key-001' });
    expect(res.status).toBe(202);
    expect(enqueueSpy).not.toHaveBeenCalled();
  });

  it('returns cached result on retry after completion', async () => {
    await redis.set(
      'idempotency:tenant-abc:key-001',
      JSON.stringify({ status: 'complete', result: { notificationId: 'ntf-xyz' } }),
      'EX', 86400
    );
    const res = await sendRequest({ idempotencyKey: 'key-001' });
    expect(res.status).toBe(200);
    expect(res.body.notificationId).toBe('ntf-xyz');
  });

  it('enqueues independently for different idempotency keys', async () => {
    const res1 = await sendRequest({ idempotencyKey: 'key-aaa' });
    const res2 = await sendRequest({ idempotencyKey: 'key-bbb' });
    expect(res1.status).toBe(202);
    expect(res2.status).toBe(202);
  });

  it('enqueues independently for same key on different tenants', async () => {
    const res1 = await sendRequest({ tenantId: 'tenant-aaa', idempotencyKey: 'key-001' });
    const res2 = await sendRequest({ tenantId: 'tenant-bbb', idempotencyKey: 'key-001' });
    expect(res1.status).toBe(202);
    expect(res2.status).toBe(202);
  });
});
```

All five tests passed. The delivery engineer also ran the reproduction scenario from Stage 1: introduced a 30-second worker delay, timed out the client, retried. One notification delivered. No duplicate.

---

## Stage 5: Validation evidence

The delivery engineer recorded:

- Test output (all passing)
- Manual reproduction scenario: pass (one notification, not two)
- Redis key inspection after each test scenario (confirmed TTL, status transitions)
- No regression in existing send tests

This was noted in the PR description under "Validation" before requesting review.

---

## Stage 6: Postmortem note

A lightweight postmortem note was added to the tracking issue (not a full postmortem — this was a bug, not an incident with customer impact beyond duplicate notifications). Format drawn from [`templates/incident-postmortem.md`](../templates/incident-postmortem.md).

**Timeline:**

- T+0: First duplicate report received from a tenant.
- T+7d: Second and third reports received. Pattern identified (all involved timeouts + retries).
- T+8d: Reliable reproduction established.
- T+9d: Root cause confirmed (TTL mismatch).
- T+10d: Fix designed, implemented, reviewed, and merged.
- T+11d: Deployed to production. No further duplicate reports.

**Root cause:** Idempotency key TTL (5s, in-memory) was shorter than realistic caller timeout windows, and state was not shared across Beacon instances. Retried requests after TTL expiry were treated as new requests.

**Why it wasn't caught earlier:** The idempotency key feature was added before Beacon was deployed at scale. At low volume, callers rarely timed out. As throughput increased, timeout-then-retry became more common.

**Fix:** Redis-backed idempotency keys with 24-hour TTL and a `processing` state that prevents re-enqueue during worker execution.

**Preventive measures:**

- Added a test that explicitly simulates timeout-then-retry against the full handler+worker stack.
- Added monitoring: `beacon.idempotency.cache_hit` and `beacon.idempotency.duplicate_blocked` counters.
- Documented idempotency key behavior in the Beacon API reference with explicit guidance on TTL expectations for callers.

---

## What AI did vs. what the human did

| Step | AI did | Human did |
|------|--------|-----------|
| Reproduce | Nothing | Designed both reproduction attempts; established reliable repro |
| Root cause | Traced execution path from handler to worker | Identified TTL mismatch by reading the idempotency implementation; confirmed root cause |
| Fix design | Nothing | Evaluated two options and chose Redis-backed approach |
| Implementation | Generated Redis storage functions | Wrote handler integration; reviewed AI output for correctness |
| Tests | Nothing | Wrote all five regression tests; ran manual reproduction check |
| Postmortem | Nothing | Wrote postmortem note; identified preventive measures |

The investigation required reading code, forming hypotheses, and verifying them. AI was useful for tracing the execution path — a task that is mechanical given the code — but the actual root cause required the human to look at the idempotency implementation directly and notice the TTL number. AI did not surface this.
