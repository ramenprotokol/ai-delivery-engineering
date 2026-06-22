# Testing Strategy

Tests are how you know a system does what you think it does. In AI-assisted delivery, tests carry extra weight: AI generates code quickly, and quickly generated code contains subtle errors. A disciplined test strategy is the primary mechanism for catching those errors before they reach production.

This document covers how tests are structured, when to use each type, how to approach AI-generated code specifically, and what the CI gates enforce.

---

## The Test Pyramid

The pyramid describes the distribution of test types. Most tests should be fast, isolated, and cheap. Fewer tests should be slow, integrated, and expensive. This is not a rule about counts — it is a rule about where confidence comes from.

```text
        /\
       /  \      E2E / smoke
      /----\
     /      \    Integration
    /--------\
   /          \  Unit
  /____________\
```

### Unit Tests

**What they test:** A single function, class, or module in isolation. All external dependencies are mocked or stubbed.

**When to write them:** For every function that transforms data, validates input, makes a decision, or computes a result. If a function has a branch (if/else, switch, try/catch), each branch needs a test.

**What they are not for:** Proving that two components work together, or that an HTTP endpoint returns the right response.

**Beacon example — validating a message payload:**

```typescript
// src/validation/message.test.ts

describe('validateMessagePayload', () => {
  it('accepts a valid email payload', () => {
    const result = validateMessagePayload({
      channel: 'email',
      to: 'recipient@example.invalid',
      subject: 'Test',
      body: 'Hello',
    });
    expect(result.valid).toBe(true);
  });

  it('rejects a payload missing the "to" field', () => {
    const result = validateMessagePayload({
      channel: 'email',
      subject: 'Test',
      body: 'Hello',
    });
    expect(result.valid).toBe(false);
    expect(result.errors).toContain('"to" is required');
  });

  it('rejects an unsupported channel', () => {
    const result = validateMessagePayload({
      channel: 'fax',
      to: 'someone@example.invalid',
      body: 'Hello',
    });
    expect(result.valid).toBe(false);
    expect(result.errors).toContain('"fax" is not a supported channel');
  });
});
```

Run time: milliseconds. No network, no database, no side effects.

### Integration Tests

**What they test:** Two or more real components working together — typically an HTTP handler and its downstream dependencies (database, queue, external client), using real or in-process implementations where possible.

**When to write them:** For every API route, every service method that calls an external system, and every data transformation that spans more than one module.

**What they are not for:** Exhaustive branching. Leave that to unit tests. Integration tests cover the realistic request-response cycle.

**Beacon example — testing the rate limit middleware on the send endpoint:**

```typescript
// src/routes/send.integration.test.ts

describe('POST /v1/messages/send — rate limiting', () => {
  beforeEach(() => resetRateLimitStore());

  it('allows requests under the limit', async () => {
    const response = await sendRequest({ apiKey: 'key-001' });
    expect(response.status).toBe(200);
  });

  it('returns 429 when the limit is exceeded', async () => {
    await exhaust1000Requests({ apiKey: 'key-001' });
    const response = await sendRequest({ apiKey: 'key-001' });
    expect(response.status).toBe(429);
  });

  it('includes Retry-After in the 429 response', async () => {
    await exhaust1000Requests({ apiKey: 'key-001' });
    const response = await sendRequest({ apiKey: 'key-001' });
    expect(response.headers['retry-after']).toBeDefined();
  });

  it('rate limit is per API key, not global', async () => {
    await exhaust1000Requests({ apiKey: 'key-001' });
    const response = await sendRequest({ apiKey: 'key-002' });
    expect(response.status).toBe(200);
  });
});
```

This test uses a real in-process HTTP server and a real (in-memory) rate limit store. It does not mock the middleware. If the middleware is broken, these tests fail.

### End-to-End / Smoke Tests

**What they test:** The full system path — real HTTP, real database, real queue, real downstream. Proves the system is alive and the happy path works.

**When to run them:** On every deployment to a staging environment, and as a post-deploy smoke check in production.

**What they are not for:** Comprehensive coverage. E2E tests are slow and brittle at scale. Use them to confirm critical paths work, not to cover every edge case.

**Beacon example — full delivery smoke test:**

```bash
#!/bin/bash
# scripts/smoke-test.sh
# Run against staging after every deploy.

API_KEY="${BEACON_SMOKE_TEST_KEY}"
BASE_URL="${BEACON_BASE_URL:-https://beacon-staging.example.invalid}"

echo "--- Beacon smoke test ---"

RESPONSE=$(curl -s -w "\n%{http_code}" \
  -X POST "$BASE_URL/v1/messages/send" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"channel":"email","to":"smoke@example.invalid","subject":"Smoke","body":"ok"}')

HTTP_STATUS=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | head -1)

if [ "$HTTP_STATUS" != "200" ]; then
  echo "FAIL: expected 200, got $HTTP_STATUS"
  echo "$BODY"
  exit 1
fi

DELIVERY_ID=$(echo "$BODY" | jq -r '.deliveryId')
if [ -z "$DELIVERY_ID" ] || [ "$DELIVERY_ID" = "null" ]; then
  echo "FAIL: no deliveryId in response"
  exit 1
fi

echo "PASS: deliveryId=$DELIVERY_ID"
```

---

## Testing AI-Generated Code

AI tools generate code fast. Fast code generation produces fast test coverage — and that is the problem. AI-written tests for AI-written code tend to share the same misunderstandings, producing tests that pass trivially and catch nothing.

### Rule 1: Write or verify tests yourself

When AI generates an implementation, the delivery engineer must either:

- Write the tests manually, without AI assistance, OR
- Read every AI-generated test line by line and verify it would actually catch a regression

"AI wrote the tests and they pass" is not a validation statement. It is a description of a process that may have produced nothing useful.

### Rule 2: Test the behavior, not the code

AI-generated code is often structurally plausible but behaviorally wrong. Tests must assert on outputs and side effects, not on internal structure.

**Weak test (tests the code, not the behavior):**

```typescript
it('calls rateLimitMiddleware', () => {
  expect(rateLimitMiddleware).toHaveBeenCalled();
});
```

**Strong test (tests the behavior):**

```typescript
it('returns 429 with Retry-After when the rate limit is exceeded', async () => {
  await exhaust1000Requests({ apiKey: 'key-001' });
  const response = await sendRequest({ apiKey: 'key-001' });
  expect(response.status).toBe(429);
  expect(response.headers['retry-after']).toMatch(/^\d+$/);
});
```

If the rate limiting logic is completely wrong but `rateLimitMiddleware` was called, the first test passes. The second test fails — which is the right outcome.

### Rule 3: Make AI-written tests fail first

Before accepting any test (AI-generated or otherwise), confirm it can fail. Introduce a deliberate bug — return the wrong status code, skip the header — and run the test. If it still passes, the test is not testing what it claims.

---

## Regression and Characterization Tests

### Regression tests

When a bug is found and fixed, a test must be added that would have caught it. The test documents the specific failure mode and prevents recurrence.

**Beacon example.** A bug is found where `POST /v1/messages/send` with a missing `body` field returns `500` instead of `400`. After the fix:

```typescript
it('returns 400 when body field is missing', async () => {
  const response = await sendRequest({
    channel: 'email',
    to: 'test@example.invalid',
    subject: 'No body',
    // body intentionally omitted
  });
  expect(response.status).toBe(400);
  expect(response.body.error).toMatch(/body.*required/i);
});
```

This test now lives in the suite permanently. If the validation is accidentally removed in a future refactor, the test fails.

### Characterization tests

When working with existing code that has no tests (common when AI refactors legacy code), write characterization tests first. Characterization tests capture the current behavior — not the desired behavior — so that a refactor does not silently change behavior.

Process:

1. Run the existing code against a set of real inputs.
2. Record the outputs.
3. Write tests that assert the recorded outputs.
4. Refactor.
5. Confirm the characterization tests still pass.

If a characterization test fails after a refactor, you changed observable behavior. That may be intentional — but it must be a conscious decision, not a surprise.

---

## Coverage Philosophy

Coverage percentage is a proxy metric. It is useful as a floor (below 60% unit coverage on a service layer is a red flag) but useless as a ceiling (100% coverage does not mean the tests are meaningful).

What meaningful coverage actually looks like:

- Every public function has at least one test for the expected happy path.
- Every validation rule has a test for a valid input and a test for each invalid case.
- Every error path that changes the response (status code, error body) has a test.
- Every integration point (database, queue, external API) has a test that exercises the real call, not a mock that returns what you told it to return.

What coverage cannot tell you:

- Whether the assertions are correct.
- Whether the test would catch a real regression.
- Whether the behavior the test covers is the behavior the system is supposed to have.

---

## CI Gates

No PR merges without passing CI. The CI pipeline enforces:

```yaml
# .github/workflows/ci.yml (excerpt)
jobs:
  validate:
    steps:
      - run: npm run lint          # must exit 0
      - run: npm run typecheck     # must exit 0
      - run: npm test -- --ci      # must exit 0, no interactive prompts
      - run: npm run build         # must produce build artifacts
```

Flaky tests that fail intermittently are treated as broken tests. They are fixed or disabled within one working day of detection. A flaky test that is left to pass-on-retry is noise in the signal — it erodes trust in the CI result and trains engineers to ignore failures.

### Flaky test handling

When a test is identified as flaky:

1. Disable it immediately (mark as `.skip` or equivalent) and open a ticket.
2. Investigate the root cause: timing issue, external dependency, shared state, non-deterministic data.
3. Fix the root cause, not the symptom. Do not add retries.
4. Re-enable the test and monitor for two CI runs before closing the ticket.

Retrying flaky tests in CI is forbidden. It hides real problems and produces a false passing signal.

---

## Test File Conventions

```text
src/
  routes/
    send.ts
    send.test.ts          # unit tests for send route handler
    send.integration.test.ts  # integration tests for the route
  validation/
    message.ts
    message.test.ts
  services/
    delivery.ts
    delivery.test.ts
tests/
  smoke/
    smoke-test.sh         # post-deploy smoke test script
```

Test files live next to the code they test. Integration tests are co-located but named to distinguish them from unit tests. Smoke tests live separately because they run against deployed environments, not local code.

---

## Related

- [validation-framework.md](validation-framework.md) — Validation levels, evidence requirements, the validation matrix
- [../checklists/testing.md](../checklists/testing.md) — Pre-merge testing checklist
- [../checklists/ai-code-review.md](../checklists/ai-code-review.md) — How to review AI-generated tests specifically
- [release-readiness.md](release-readiness.md) — What test state is required before a release
