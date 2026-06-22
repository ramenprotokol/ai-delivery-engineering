# Feature Delivery Walkthrough: Rate Limiting for Beacon's Send API

This is a synthetic end-to-end demonstration of delivering a non-trivial feature using the methodology in [`docs/delivery-lifecycle.md`](../docs/delivery-lifecycle.md). Every system, metric, and person is fictional.

---

## Feature summary

**System:** Beacon (fictional transactional notification API)
**Feature:** Per-tenant rate limiting on `POST /v1/send`
**Trigger:** Repeated incidents where a single tenant's burst traffic delayed delivery for other tenants. The on-call engineer flagged this in three consecutive incident reviews.
**Outcome target:** No single tenant can consume more than 20% of send throughput in any 10-second window. Tenants that exceed the limit receive a `429` response with a `Retry-After` header. No tenant's traffic affects another's.

---

## Stage 1: Scope

The delivery engineer read the three incident reports and extracted the pattern: the problem was not throughput — Beacon had headroom — it was fairness. One tenant's burst filled the worker queue and starved others.

**Questions resolved before design began:**

- Is the limit per-tenant or per-API-key? Per-tenant. A tenant may have multiple API keys; the limit applies to the tenant account.
- Fixed limit or configurable? Configurable per-tenant via the admin API, with a system-wide default.
- What does the caller receive on limit? `429 Too Many Requests` with `Retry-After: <seconds>` and a JSON body: `{"error": "rate_limit_exceeded", "retry_after": 5}`.
- Redis already deployed? Yes — it is already used for idempotency key storage.

**Scope boundaries (explicit):**

- In scope: rate limiting on `POST /v1/send`, per-tenant, backed by Redis sliding window.
- Out of scope: rate limiting on other endpoints, per-IP limiting, billing integration, alerting on limit hits (filed as a follow-on).

These decisions were captured as a scope note in the tracking issue before any design work started.

---

## Stage 2: Design and ADR

The delivery engineer drafted a design doc using [`templates/design-doc.md`](../templates/design-doc.md). Key decisions:

**Algorithm:** Redis sliding window counter using `ZRANGEBYSCORE` / `ZADD` / `ZREMRANGEBYSCORE`. Chosen over token bucket because it gives accurate per-window counts with no background expiry process needed, and the implementation is straightforward given the existing Redis client.

**Storage layout:**

```text
key:  rate:<tenant_id>
type: sorted set
member: <request_id>  (UUID)
score: <unix_ms timestamp>
ttl:  2x the window size (auto-expire stale sets)
```

**Failure mode:** If Redis is unavailable, rate limiting fails open. Beacon sends the notification and logs the Redis error. The reasoning: a temporary Redis outage should not cause 429s for all tenants. This is a deliberate tradeoff, documented in the ADR.

**ADR filed:** See [`examples/sample-architecture-decision-record.md`](sample-architecture-decision-record.md) — the completed ADR for this decision.

The design doc went through one review cycle. A reviewer asked whether the sliding window approach would cause thundering-herd retries if many tenants hit the limit simultaneously. The delivery engineer added the `Retry-After` header with per-tenant jitter (random offset ±2s) to mitigate this.

---

## Stage 3: Build with AI assistance

### Where AI was used

**Task 1: Redis sliding window implementation**

The delivery engineer wrote a prompt describing the algorithm and the existing Redis client interface:

```text
Context: I have a Redis client with methods get(), set(), zadd(), zrangebyscore(),
zremrangebyscore(), expire(). I need a sliding window rate limiter:
- Window: configurable, default 10 seconds
- Limit: configurable per tenant, default 1000 requests/window
- Key pattern: rate:<tenant_id>
- Returns: { allowed: bool, count: int, limit: int, reset_at: unix_ms }
- On Redis error: log and return { allowed: true, ... } (fail open)
Write the function in TypeScript. No external rate-limit libraries.
```

AI produced a working implementation. The delivery engineer reviewed it against the ADR, found that the TTL was set to the window size rather than 2x, and corrected it. The original AI output and the correction were recorded in the AI change log (see Stage 3 artifact below).

**Task 2: Middleware integration**

AI generated the Express middleware wrapper given the rate limiter function signature and the required response shape. The delivery engineer reviewed the error handling path — AI had returned a generic 500 on Redis errors rather than failing open — and rewrote that branch.

**Task 3: Test scaffolding**

AI generated the test file structure and the happy-path test cases. The delivery engineer wrote the edge cases: Redis failure behavior, window boundary behavior, jitter range on `Retry-After`.

### What the human decided

- Algorithm selection (sliding window over token bucket)
- Failure mode (fail open)
- Jitter on `Retry-After`
- TTL correction (2x window, not 1x)
- Redis error handling path rewrite in middleware
- Which test cases AI missed and needed to be written by hand

### AI change log entry (excerpt)

From the AI change log artifact (format: [`templates/ai-change-log.md`](../templates/ai-change-log.md)):

```text
## 2024-03-14 — Rate limiter core function

Tool: Claude Code
Prompt: [see above]
Output: Generated slidingWindowRateLimiter() in src/lib/rate-limit.ts
Human review: TTL set to windowMs instead of windowMs * 2. Corrected before commit.
Human review: Verified ZRANGEBYSCORE key boundaries match algorithm spec.
Approved: yes
Committed: yes
```

---

## Stage 4: Validate

The test plan was written before the implementation started (see [`examples/sample-test-plan.md`](sample-test-plan.md)).

### Test categories and results

**Unit tests — rate limiter logic**

```typescript
// Boundary: request at exactly the limit
it('allows the Nth request and blocks the N+1th', async () => {
  const limiter = createRateLimiter({ limit: 5, windowMs: 10_000 });
  for (let i = 0; i < 5; i++) {
    const result = await limiter.check('tenant-abc');
    expect(result.allowed).toBe(true);
  }
  const result = await limiter.check('tenant-abc');
  expect(result.allowed).toBe(false);
  expect(result.count).toBe(6);
});

// Redis failure: fail open
it('allows the request when Redis throws', async () => {
  redisClient.zadd.mockRejectedValue(new Error('connection refused'));
  const result = await limiter.check('tenant-abc');
  expect(result.allowed).toBe(true);
});

// Tenant isolation: one tenant's count does not affect another
it('counts requests independently per tenant', async () => {
  for (let i = 0; i < 5; i++) await limiter.check('tenant-aaa');
  const result = await limiter.check('tenant-bbb');
  expect(result.allowed).toBe(true);
});
```

All three passed. AI had not generated the Redis failure test or the tenant isolation test — both were written by the delivery engineer.

**Integration tests — middleware + API**

Tested with a local Redis instance:

| Scenario | Expected | Result |
|----------|----------|--------|
| 10 requests under limit | All 200 | Pass |
| 11th request over limit | 429 + `Retry-After` header | Pass |
| `Retry-After` value in range | Default window − elapsed + jitter | Pass |
| Redis stopped mid-test | Requests continue with 200 | Pass |
| Two tenants at limit simultaneously | Neither blocks the other | Pass |

**Load test (synthetic)**

Ran a local load test with 3 concurrent tenants, each sending bursts of 50 requests/second over 60 seconds. Confirmed:

- No cross-tenant limit bleed
- Redis key TTLs expiring correctly (verified via `redis-cli TTL`)
- p99 latency increase from rate-limit check: 4ms (acceptable)

### Validation decisions the human made

- Decided "fail open" needed an explicit test, not just a comment in code.
- Reviewed the load test numbers and judged the 4ms p99 addition acceptable given the incident history.
- Signed off on the test plan coverage as sufficient before approving the PR.

---

## Stage 5: Review

The PR was opened using the pull request template ([`.github/PULL_REQUEST_TEMPLATE.md`](../.github/PULL_REQUEST_TEMPLATE.md)).

The delivery engineer ran the pre-merge checklist ([`checklists/pre-merge.md`](../checklists/pre-merge.md)) and the AI code review checklist ([`checklists/ai-code-review.md`](../checklists/ai-code-review.md)) before requesting review.

**AI code review checklist findings (self-review before PR):**

- AI-generated code: rate limiter core function and middleware shell
- Human-modified sections: TTL correction, error handling path, edge-case tests
- Sensitive paths reviewed: Redis key pattern (no PII in keys), `Retry-After` value construction (no information leak)
- No AI-generated code was merged without human read-through

**Reviewer feedback:**

- Asked about behavior when a tenant's configured limit is reduced mid-window. Answer: the window's existing count is respected; the new limit takes effect on the next window. This was added to the ADR as a clarifying note.
- Suggested adding a log line on every 429 response (not just Redis errors) so limit-hit rate is observable. The delivery engineer agreed and added it.

The delivery engineer did not approve their own PR. A second engineer reviewed and approved.

---

## Stage 6: Release

The release plan was prepared using [`templates/release-plan.md`](../templates/release-plan.md) (see [`examples/sample-release-plan.md`](sample-release-plan.md) for the completed version).

**Deployment strategy:** Feature flag. Rate limiting defaults to off for all tenants. Enabled per-tenant via the admin API. This allows gradual rollout without a staged deployment.

**Go / no-go check (recorded in the release plan):**

| Check | Status |
|-------|--------|
| All tests passing in CI | Go |
| Redis connection verified in staging | Go |
| Default limit configured and reviewed | Go |
| Feature flag confirmed off by default | Go |
| Rollback procedure documented | Go |
| On-call informed of deployment | Go |

Approver: the second engineer who reviewed the PR. Decision: Go.

**Rollout sequence:**

1. Deploy to production with rate limiting disabled (feature flag off).
2. Enable on a single internal-test tenant. Monitor for 30 minutes.
3. Enable on three pilot tenants. Monitor for 2 hours.
4. Enable system-wide default. Monitor for 24 hours.
5. Close the tracking issue.

---

## Stage 7: Operate

**Monitoring added:**

- `beacon.rate_limit.hits` — counter, tagged by tenant_id
- `beacon.rate_limit.redis_errors` — counter (fail-open events)
- `beacon.rate_limit.check_duration_ms` — histogram

**Runbook section added** to the Beacon runbook (format: [`templates/runbook.md`](../templates/runbook.md)):

> **Rate limit hit spike on a tenant**
>
> 1. Check `beacon.rate_limit.hits` tagged by tenant_id to identify which tenant.
> 2. Check whether the tenant has a non-default limit configured in the admin API.
> 3. If the burst is legitimate (tenant is scaling up), increase their limit via the admin API. No deployment needed.
> 4. If the burst looks anomalous (e.g., a client bug sending duplicate requests), contact the tenant.
> 5. If Redis errors are spiking, rate limiting is in fail-open mode — all requests are passing. Check Redis cluster health.

**Post-deployment observation (30 days):**

- Zero cross-tenant complaints since rollout.
- Three tenants hit the default limit in the first week; two were client-side bugs the tenants fixed themselves after seeing 429s; one was a legitimate scale event and the limit was increased.
- Redis fail-open events: 0.

---

## Artifacts produced

| Artifact | Location |
|----------|----------|
| Scope note | Tracking issue (internal) |
| Design doc | Tracking issue (internal) |
| ADR | [`examples/sample-architecture-decision-record.md`](sample-architecture-decision-record.md) |
| AI change log | [`templates/ai-change-log.md`](../templates/ai-change-log.md) |
| Test plan | [`examples/sample-test-plan.md`](sample-test-plan.md) |
| Pre-merge checklist | [`checklists/pre-merge.md`](../checklists/pre-merge.md) |
| AI code review checklist | [`checklists/ai-code-review.md`](../checklists/ai-code-review.md) |
| Release plan | [`examples/sample-release-plan.md`](sample-release-plan.md) |
| Runbook section | Beacon runbook |

---

## What AI did vs. what the human did

| Stage | AI did | Human did |
|-------|--------|-----------|
| Scope | Nothing | Analyzed incident reports, identified the fairness problem, defined boundaries |
| Design | Nothing | Algorithm selection, failure mode decision, jitter design |
| Build | Generated rate limiter core, middleware shell, test scaffolding | Corrected TTL bug, rewrote error handling, wrote edge-case tests |
| Validate | Nothing | Designed test plan, interpreted load test results, approved coverage |
| Review | Nothing | Ran checklists, responded to reviewer feedback, approved PR |
| Release | Nothing | Wrote release plan, made go/no-go call, sequenced rollout |
| Operate | Nothing | Defined metrics, wrote runbook section, responded to post-deploy events |

AI accelerated Stage 3 by roughly half a day. Every other stage was fully human-driven. The TTL bug and the error handling bug in AI-generated code confirm why human review is not optional.
