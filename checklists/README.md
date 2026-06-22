# Checklists

Checklists encode the repeatable parts of delivery so that judgment is spent where it matters — on the edge cases, the architectural tradeoffs, and the decisions that don't have a box to tick.

Every checklist here is gate-oriented: it defines a specific decision point (merge, release, AI code review, etc.) and the conditions that must be true to pass that gate. The items are concrete and binary. If an item cannot be checked because it does not apply, note why — don't silently skip it.

## Index

| Checklist | When to use | Purpose |
|---|---|---|
| [pre-merge.md](pre-merge.md) | Before merging any pull request | Confirm the change is correct, tested, documented, and safe to merge |
| [release-readiness.md](release-readiness.md) | Before promoting to production | Go/no-go gate: validation evidence, monitoring, rollback, approver sign-off |
| [ai-code-review.md](ai-code-review.md) | When reviewing AI-generated code | Specific checks for hallucinations, logic, security, and reviewer ownership |
| [security-review.md](security-review.md) | Before any security-sensitive change ships | Auth, secrets, input validation, dependency surface, blast radius |
| [production-readiness.md](production-readiness.md) | Before a new service or major feature goes live | Infrastructure, observability, on-call, runbook, capacity, SLOs |
| [documentation.md](documentation.md) | Before closing any ticket or PR | Confirm the right things were written, updated, or explicitly deferred |
| [testing.md](testing.md) | During test planning and execution | Coverage, types, environments, pass criteria, evidence |

## How to use these

1. Copy the relevant checklist into your PR description, ticket, or release record before you start.
2. Work through items in order. Each section has a logical dependency on the ones before it.
3. Check items only when you have verified them — not when you expect them to be true.
4. For items marked `[if applicable]`: document why they do not apply rather than deleting the line.
5. A checklist with unresolved items is a blocked gate. Escalate or defer explicitly — don't ship around it.

## Relationship to other documents

- Checklists implement the gates described in [docs/delivery-lifecycle.md](../docs/delivery-lifecycle.md).
- The validation philosophy behind them is in [docs/validation-framework.md](../docs/validation-framework.md).
- Templates for the artifacts these checklists reference (ADRs, release plans, postmortems) live in [templates/](../templates/README.md).
- Worked examples showing checklists applied to real scenarios are in [examples/](../examples/README.md).

## Contributing a new checklist

See [CONTRIBUTING.md](../CONTRIBUTING.md). New checklists must have a clear decision-point owner, binary items (not vague guidance), and a cross-link from this index.
