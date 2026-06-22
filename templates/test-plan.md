# Test Plan: [Feature or Change Name]

<!-- Title should match the design doc or ticket it covers. Example: "Test Plan: Beacon Idempotent Retry Handling" -->

| Field | Value |
|---|---|
| **Status** | <!-- Draft / Active / Complete --> |
| **Author(s)** | <!-- Roles only --> |
| **Linked Design Doc** | <!-- Relative path or URL --> |
| **Linked Release Plan** | <!-- Relative path or URL, if available --> |
| **Created** | <!-- YYYY-MM-DD --> |
| **Last Updated** | <!-- YYYY-MM-DD --> |
| **AI Assistance** | <!-- Example: "Claude Code generated initial test case table from design doc; delivery engineer reviewed, removed duplicates, and added edge cases." --> |
| **Approved By** | <!-- Role and date --> |

---

## Scope

<!-- What is in scope for this test plan? Be specific about which system, component, version, or feature is being tested. -->

**In scope:**

- <!-- Component or behavior being tested -->
- <!-- Another component or behavior -->

**Out of scope:**

- <!-- What is explicitly not being tested here, and why -->

## Test Objectives

<!-- What must this test effort confirm before the change is considered shippable? Write objectives as verifiable statements. -->

- Verify that <!-- specific behavior or contract holds -->
- Confirm that <!-- edge case or error condition is handled correctly -->
- Validate that <!-- performance or load characteristic is met -->
- Ensure that <!-- existing behavior is not regressed -->

## Environments

| Environment | Purpose | Notes |
|---|---|---|
| Local / Dev | Unit and integration development | Individual developer machines |
| Staging | Integration and E2E testing | Mirrors production configuration; uses synthetic data only |
| Production (post-deploy) | Smoke test and monitoring | Limited to read operations and canary traffic where possible |

## Test Data

<!-- Describe the test data strategy. All test data must be synthetic — no real user data in any environment below production. -->

- <!-- What data is needed and where it comes from (fixtures, factories, seeds) -->
- <!-- Any special data conditions required (empty state, at-limit, corrupted input) -->
- <!-- How test data is cleaned up after runs -->

## Entry Criteria

<!-- The change must meet these conditions before testing begins. -->

- [ ] Code is reviewed and merged to the test branch
- [ ] Staging environment is deployed and healthy
- [ ] Test data is seeded
- [ ] <!-- Any other prerequisite -->

## Exit Criteria

<!-- Testing is complete when ALL of these are true. -->

- [ ] All test cases in the table below have a recorded result (Pass, Fail, or Skipped with reason)
- [ ] All failures are resolved or accepted with documented rationale
- [ ] No P0 or P1 defects are open
- [ ] Sign-off obtained from approver listed above

---

## Test Cases

<!-- Fill in one row per test case. Add rows as needed. Keep IDs sequential. -->

| ID | Type | Scenario | Preconditions | Steps | Expected Result | Actual Result | Status |
|---|---|---|---|---|---|---|---|
| TC-001 | Unit | <!-- What is being tested at unit level --> | <!-- Setup required --> | 1. <!-- step --> 2. <!-- step --> | <!-- What should happen --> | <!-- Filled during execution --> | <!-- Pass / Fail / Skip --> |
| TC-002 | Unit | <!-- --> | <!-- --> | 1. <!-- --> | <!-- --> | | |
| TC-003 | Integration | <!-- Happy-path end-to-end through two components --> | <!-- --> | 1. <!-- --> 2. <!-- --> 3. <!-- --> | <!-- --> | | |
| TC-004 | Integration | <!-- Error path — upstream returns 5xx --> | <!-- --> | 1. <!-- --> | <!-- --> | | |
| TC-005 | E2E | <!-- Full flow from API call to observable side effect --> | <!-- Staging deployed --> | 1. <!-- --> 2. <!-- --> | <!-- --> | | |
| TC-006 | Manual | <!-- Edge case best caught by a human looking at the UI or logs --> | <!-- --> | 1. <!-- --> | <!-- --> | | |
| TC-007 | Regression | <!-- A previously shipped behavior that must not break --> | <!-- --> | 1. <!-- --> | <!-- --> | | |

<!-- Add more rows as needed. Group by type if the table grows large. -->

---

## Test Types Detail

### Unit Tests

<!-- What is covered by automated unit tests? Which files, functions, or modules? What framework is used? How are they run? -->

```bash
# Example: how to run the unit suite locally
# npm test -- --testPathPattern=src/delivery
```

### Integration Tests

<!-- What is covered by integration tests? What external dependencies are stubbed vs. real? What framework? -->

```bash
# Example: how to run integration tests against staging
```

### End-to-End Tests

<!-- What scenarios are covered? What tool drives them (Playwright, Cypress, custom scripts)? What environment do they run against? -->

### Manual / Exploratory Testing

<!-- What requires human judgment that automated tests cannot cover? Who performs it? How is the result recorded? -->

---

## Defect Management

| Severity | Definition | Resolution Requirement |
|---|---|---|
| P0 — Critical | System is down or data is corrupted | Must be fixed before release |
| P1 — High | Core functionality broken, no workaround | Must be fixed before release |
| P2 — Medium | Functionality degraded, workaround exists | Fixed before release or accepted with documented rationale |
| P3 — Low | Minor issue, cosmetic, or edge case | May be deferred to follow-up |

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| <!-- Test environment instability --> | <!-- Low/Med/High --> | <!-- Low/Med/High --> | <!-- Keep a rollback path for the staging deploy --> |
| <!-- Insufficient edge case coverage --> | <!-- --> | <!-- --> | <!-- Pair review of test cases against design doc before execution begins --> |

## Sign-off

| Role | Decision | Date |
|---|---|---|
| <!-- Delivery engineer --> | Ready to release / Not ready | <!-- YYYY-MM-DD --> |
| <!-- Reviewer or lead --> | Approved / Changes required | <!-- YYYY-MM-DD --> |

---

*Related: [Design Doc template](design-doc.md) · [Release Plan template](release-plan.md) · [Testing Strategy](../docs/testing-strategy.md) · [AI Code Review Checklist](../checklists/ai-code-review.md)*
