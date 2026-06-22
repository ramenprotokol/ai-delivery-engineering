# Pre-Merge Checklist

Use this before merging any pull request, regardless of size. Copy it into the PR description and check items as you go. A PR with unchecked items is not ready to merge.

> **Who is responsible:** The author prepares the checklist. The reviewer verifies it. The merger owns the final state.

---

## 1. Code Review

- [ ] At least one human reviewer has approved this PR — not just the author reviewing their own work
- [ ] Every reviewer comment has been resolved or explicitly acknowledged (note the reason if not actioned)
- [ ] The reviewer understood the intent of the change, not just the mechanics of the diff

## 2. AI-Generated Code

- [ ] AI-generated sections are identified (inline comment or PR description noting which tool was used)
- [ ] Each AI-generated section has been read and understood by a human, not just accepted
- [ ] The [ai-code-review.md](ai-code-review.md) checklist has been completed for any AI-generated code
- [ ] No hallucinated APIs, functions, or imports are present — every reference has been verified against actual documentation
- [ ] The AI change log ([templates/ai-change-log.md](../templates/ai-change-log.md)) has been updated with a record of what AI generated and what was changed

## 3. Tests

- [ ] New behavior has tests covering the happy path
- [ ] New behavior has tests covering expected failure modes and edge cases
- [ ] Existing tests that the change could affect have been reviewed and updated if needed
- [ ] All tests pass locally and in CI
- [ ] No tests have been skipped or disabled without a documented reason
- [ ] Tests actually test the behavior — they would fail if the implementation were wrong (not tautologies)

## 4. Code Quality

- [ ] Lint passes with no new warnings
- [ ] Formatter has been run and the diff is clean
- [ ] No dead code, commented-out blocks, or debug logging left in
- [ ] No hard-coded values that belong in config or environment variables
- [ ] Dependencies added in this PR are intentional and the version is pinned

## 5. Secrets and Security

- [ ] No secrets, credentials, tokens, API keys, or personal data appear in the diff
- [ ] No new environment variables are assumed without documentation
- [ ] If this PR changes authentication, authorization, or input handling — the [security-review.md](security-review.md) checklist has been run
- [ ] `.gitignore` and `.env.example` are up to date if new config was introduced

## 6. Documentation

- [ ] Inline comments explain the *why* for any non-obvious logic
- [ ] Public API changes (endpoints, interfaces, CLIs) are reflected in the relevant docs
- [ ] If a runbook, README, or setup guide is affected by this change, it has been updated
- [ ] The [documentation.md](documentation.md) checklist has been checked if documentation is a significant part of this PR
- [ ] CHANGELOG.md has an entry (or the PR is labeled `no-changelog` with a reason)

## 7. Architecture Decisions

- [ ] If this PR introduces or changes a significant architectural decision (data model, external dependency, protocol, pattern), an ADR has been filed using [templates/architecture-decision-record.md](../templates/architecture-decision-record.md)
- [ ] If an existing ADR is superseded by this change, the old ADR has been updated to status "Superseded" and linked to the new one

## 8. Rollback and Risk

- [ ] The change can be rolled back — either by reverting the commit or toggling a feature flag
- [ ] If rollback requires a migration or coordination step, that is documented in the PR description
- [ ] The risk level of this change has been assessed: low / medium / high (note it in the PR)
- [ ] For medium or high risk: a reviewer other than the author has explicitly confirmed the rollback path

## 9. Pre-Merge Final Check

- [ ] The PR title is accurate and matches the change
- [ ] The PR description explains the *why* (not just the what) and links to any relevant ticket or ADR
- [ ] Branch is up to date with the target branch — no unresolved conflicts
- [ ] CI pipeline is green (all jobs passing, no flaky-test overrides in place)

---

**Gate:** All items above must be checked before merge. If an item does not apply, replace `[ ]` with `[N/A]` and add a one-line reason inline.
