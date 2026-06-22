# Test Plan: Beacon Rate Limiting (v1.3.0)

**Feature:** Per-tenant rate limiting for outbound notification requests  
**Version:** Beacon v1.3.0  
**Author role:** Delivery engineer  
**Reviewer role:** Backend engineer  
**Date:** 2026-04-14  
**Template:** [templates/test-plan.md](../templates/test-plan.md)

---

## 1. Scope

This plan covers functional, boundary, and failure-path testing for the rate limiter introduced in Beacon v1.3.0. The rate limiter enforces per-tenant request quotas using a sliding window counter backed by Redis.

**In scope**

- Requests accepted within quota
- Requests rejected at and beyond quota limit
- Correct HTTP 429 response shape and headers
- Quota reset after the window rolls over
- Behavior when Redis is unavailable (fail-open policy)
- Multi-instance consistency (two Beacon workers share one Redis counter)

**Out of scope**

- Downstream provider rate limits (tested separately in provider integration tests)
- Quota administration UI (not yet built)
- Load / performance testing (covered in [docs/testing-strategy.md](../docs/testing-strategy.md) under load testing)

---

## 2. Environments

| Environment | Purpose | Rate limit config |
|---|---|---|
| Local (Docker Compose) | Developer iteration | 10 req / 60 s per tenant |
| Staging | Pre-merge validation | 100 req / 60 s per tenant (mirrors production values) |
| Production | Post-deploy smoke test | 500 req / 60 s per tenant (live values, smoke only — 3 requests) |

All test cases in section 4 run against **staging** unless noted.

---

## 3. Entry Criteria

- [ ] Feature branch merged to `main` and deployed to staging.
- [ ] Redis available and healthy in staging (`redis-cli ping` returns `PONG`).
- [ ] Staging Beacon logs streaming to the observability dashboard.
- [ ] Test tenants provisioned: `tenant-a` (quota: 100/60s), `tenant-b` (quota: 10/60s).
- [ ] Previous test run results cleared (flush staging Redis test keys).

---

## 4. Test Cases

### 4.1 Happy Path

| ID | Description | Steps | Expected result | Actual result | Pass/Fail |
|---|---|---|---|---|---|
| RL-01 | Request within quota is accepted | Send 1 POST `/v1/send` with `tenant-a` key | HTTP 200; `X-RateLimit-Remaining: 99`; `X-RateLimit-Limit: 100` | — | — |
| RL-02 | Sequential requests within quota are all accepted | Send 50 requests in sequence with `tenant-a` key over 30 s | All return HTTP 200; `X-RateLimit-Remaining` decrements correctly from 99 to 50 | — | — |
| RL-03 | Response includes correct rate limit headers | Send 1 request with `tenant-b` key | Response includes `X-RateLimit-Limit: 10`, `X-RateLimit-Remaining: 9`, `X-RateLimit-Reset` (Unix timestamp within current 60 s window) | — | — |

### 4.2 Quota Boundary

| ID | Description | Steps | Expected result | Actual result | Pass/Fail |
|---|---|---|---|---|---|
| RL-04 | Request exactly at quota limit is rejected | Send 10 requests with `tenant-b` (exhausts quota), then send request 11 | First 10 return HTTP 200; request 11 returns HTTP 429 | — | — |
| RL-05 | 429 response body is correct | Send request 11 as above | Body is `{"error":"rate_limit_exceeded","retry_after":N}` where N is seconds until window reset | — | — |
| RL-06 | `Retry-After` header matches body | Send request 11 as above | `Retry-After` header value equals `retry_after` in body (±1 s) | — | — |
| RL-07 | Quota resets after window expires | Send 10 requests with `tenant-b`, wait for window reset (≤ 60 s), send 1 more | After reset, request returns HTTP 200; `X-RateLimit-Remaining: 9` | — | — |
| RL-08 | Tenants do not share quota | Exhaust `tenant-b` quota (10 requests); send 1 request with `tenant-a` | `tenant-a` request returns HTTP 200 (separate counter) | — | — |

### 4.3 Sliding Window Accuracy

| ID | Description | Steps | Expected result | Actual result | Pass/Fail |
|---|---|---|---|---|---|
| RL-09 | Window slides, not fixed | Send 5 requests with `tenant-b` at T+0. Wait 35 s. Send 5 more at T+35. At T+61, confirm old requests have expired, send 5 more | Third batch (T+61) accepted; remaining = 5 (only T+35 requests still in window) | — | — |
| RL-10 | No rounding errors at window boundary | Hammer 10 requests within 1 s of window reset | Exactly 10 new requests accepted; 11th rejected; no off-by-one window double-count | — | — |

### 4.4 Multi-Instance Consistency

| ID | Description | Steps | Expected result | Actual result | Pass/Fail |
|---|---|---|---|---|---|
| RL-11 | Two workers share one counter | Route 5 requests to Beacon worker-1 and 5 requests to Beacon worker-2, all with `tenant-b` | Combined total of 10 requests accepted; next request to either worker returns HTTP 429 | — | — |
| RL-12 | Worker restart does not reset counter | Send 8 requests to worker-1, restart worker-1, send 3 more to worker-1 | Requests 9 and 10 accepted; request 11 rejected (counter persisted in Redis across restart) | — | — |

### 4.5 Failure Mode — Redis Unavailable

| ID | Description | Steps | Expected result | Actual result | Pass/Fail |
|---|---|---|---|---|---|
| RL-13 | Fail-open: requests pass when Redis is down | Stop Redis in staging, send a request with any tenant key | HTTP 200 returned; Beacon logs `[WARN] rate_limiter: redis unavailable, failing open` | — | — |
| RL-14 | Error is observable | As above | Metric `beacon_ratelimiter_redis_error_total` increments; alert fires within 2 minutes | — | — |
| RL-15 | Quota enforcement resumes when Redis recovers | Restore Redis, send requests with `tenant-b` up to quota | Normal 429 behavior resumes; no stale counter state from the outage period | — | — |

### 4.6 Edge Cases

| ID | Description | Steps | Expected result | Actual result | Pass/Fail |
|---|---|---|---|---|---|
| RL-16 | Unknown tenant key is rejected before rate limiter | Send request with invalid API key | HTTP 401 (auth rejects before rate limiter runs; no Redis write occurs) | — | — |
| RL-17 | Malformed request does not increment counter | Send request with missing required body field | HTTP 400; counter for tenant not incremented (verify with subsequent valid requests showing full quota) | — | — |
| RL-18 | Large burst within 1 second | Send 10 concurrent requests with `tenant-b` at the same instant | Exactly 10 accepted (or fewer if race causes early reject), 0 accepted above quota; no panic or data race in logs | — | — |

---

## 5. Regression Checks

Run the existing Beacon integration test suite (`make test-integration`) after staging deploy. All previously passing tests must continue to pass. Rate limiting must not affect tenants with valid quota.

---

## 6. Exit Criteria

- All test cases in section 4 marked Pass.
- Zero new errors in staging logs attributable to the rate limiter (excluding intentional Redis failure tests).
- `make test-integration` passes with no regressions.
- Metrics `beacon_ratelimiter_accepted_total` and `beacon_ratelimiter_rejected_total` visible in the observability dashboard and incrementing as expected during testing.
- Production smoke test (RL-01 equivalent, 3 requests with a real tenant key) returns HTTP 200.

---

## 7. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Redis counter drift under concurrent load | Low | Medium | Lua script enforces atomic increment-and-check; tested in RL-18 |
| Window boundary off-by-one | Medium | Medium | RL-10 specifically targets this; sliding window implementation reviewed against unit tests |
| Fail-open policy causes quota bypass during outage | Accepted | Medium | Design decision; logged and alerted (RL-14); revisit if abuse patterns emerge |
| Rate limit headers absent on 429 | Low | Low | RL-05 and RL-06 explicitly check header presence and values |

---

## 8. AI Assistance Note

The delivery engineer used Claude Code to generate the initial test case table from the feature specification and to identify boundary conditions (window sliding, concurrent requests, Redis failure paths). The engineer reviewed each generated case, removed two duplicates, added RL-09 (sliding window accuracy) and RL-12 (counter persistence across restart) which the AI draft omitted, and adjusted quota values to match staging configuration. Test execution and pass/fail determination are performed by the engineer.

---

## 9. Sign-Off

| Role | Sign-off | Date |
|---|---|---|
| Delivery engineer (author) | — | — |
| Backend engineer (reviewer) | — | — |
| Engineering manager (approver) | — | — |

---

## References

- [docs/testing-strategy.md](../docs/testing-strategy.md)
- [docs/validation-framework.md](../docs/validation-framework.md)
- [checklists/testing.md](../checklists/testing.md)
- [examples/sample-release-plan.md](sample-release-plan.md)
- [templates/test-plan.md](../templates/test-plan.md)
