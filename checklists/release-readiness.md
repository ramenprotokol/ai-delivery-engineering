# Release Readiness Checklist

This is a go/no-go gate. Complete it before promoting any change to production. Every item must be explicitly checked or marked not-applicable with a reason. A single unchecked item is a hold — escalate or resolve before proceeding.

> **Who owns this gate:** A named release approver signs off at the bottom. The on-call engineer is notified before release begins. The delivery engineer is responsible for completing this checklist before asking for approval.

---

## 1. Validation and Testing

- [ ] All automated tests pass in CI — no skips, no known flaky overrides masking real failures
- [ ] Manual validation has been performed against the acceptance criteria defined in the release plan
- [ ] Evidence of validation exists (CI run link, test report, or screen recording) and is linked in the release record
- [ ] Any changes to the Beacon API, Atlas CLI, or Orchard integrations have been tested against a staging environment that matches production configuration
- [ ] Performance impact has been assessed — response time, throughput, and resource usage are within acceptable range
- [ ] If a test plan was written ([templates/test-plan.md](../templates/test-plan.md)), all planned test cases have a pass/fail result recorded

## 2. Code and Change Quality

- [ ] The [pre-merge.md](pre-merge.md) checklist was completed for all PRs in this release
- [ ] The [ai-code-review.md](ai-code-review.md) checklist was completed for any AI-generated code in this release
- [ ] No known bugs in the release scope are being shipped without an explicit, documented accept decision
- [ ] CHANGELOG.md is updated with accurate entries for this release version

## 3. Monitoring and Alerting

- [ ] Application logs are flowing to the expected destination and are readable
- [ ] Key metrics for this change are being tracked (error rate, latency, throughput, or business metric depending on the change)
- [ ] Alerts exist for conditions that would indicate the release is failing — and they have been tested or reviewed recently
- [ ] Dashboards relevant to this change are accessible and current
- [ ] A baseline of pre-release metrics has been recorded so post-release comparison is possible

## 4. Rollback Plan

- [ ] A rollback procedure is documented in the release plan ([templates/release-plan.md](../templates/release-plan.md))
- [ ] The rollback has been tested (dry-run, staging verification, or prior successful use) — not just written down
- [ ] If rollback requires a database migration reversal, the reverse migration exists and has been tested
- [ ] If rollback requires coordination with another team or system, that team is aware and available during the release window
- [ ] Estimated rollback time is documented — the on-call engineer knows how long it will take

## 5. Feature Flags and Progressive Delivery

- [ ] `[if applicable]` New features are behind a feature flag — the flag is confirmed off in production before release begins
- [ ] `[if applicable]` Canary or staged rollout percentage is configured and the plan for full rollout is documented
- [ ] `[if applicable]` The flag management system is accessible to the on-call engineer without requiring a deployment to change flag state
- [ ] `[if applicable]` Traffic routing or percentage thresholds have been reviewed against current load

## 6. Dependencies

- [ ] All runtime dependencies (libraries, services, infrastructure) are pinned to known-good versions
- [ ] A dependency vulnerability scan has been run and any new findings have been triaged
- [ ] External service dependencies (third-party APIs, message queues, databases) have been confirmed operational in staging
- [ ] Infrastructure changes (config, environment variables, secrets) are deployed ahead of or alongside this release, not after

## 7. Documentation and Communication

- [ ] User-facing changes are documented (API docs, changelogs, release notes) if end users or integrators will be affected
- [ ] Internal runbook ([templates/runbook.md](../templates/runbook.md)) is updated to reflect any operational changes in this release
- [ ] The release plan is finalized and accessible to everyone involved in the release
- [ ] Any deprecation notices, breaking changes, or migration instructions are communicated to affected parties before release

## 8. On-Call and Team Readiness

- [ ] The on-call engineer knows the release is happening and has the release plan and rollback procedure
- [ ] The release window is scheduled at a time when the on-call engineer can monitor for at least one hour post-deploy
- [ ] Escalation contacts are documented for this release (who to reach if on-call needs backup)
- [ ] No other high-risk releases or infrastructure changes are scheduled in the same window

## 9. Approver Sign-Off

- [ ] The release plan has been reviewed by the named approver
- [ ] The approver has confirmed they understand the change, the risk, and the rollback plan
- [ ] Any open concerns raised during review have been resolved or formally accepted
- [ ] The approver has explicitly authorized the release to proceed

---

**Release decision:**

| Field | Value |
|---|---|
| Release version | |
| Release window | |
| Delivery engineer | |
| Named approver | |
| On-call engineer | |
| Go / No-Go | **GO** / **NO-GO** |
| Sign-off time (UTC) | |

**Hold reasons (if No-Go):**

_Document each unresolved item and the owner responsible for resolving it._

---

> See also: [docs/release-readiness.md](../docs/release-readiness.md) for the reasoning behind these gates, and [templates/release-plan.md](../templates/release-plan.md) for the release planning template.
