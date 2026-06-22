# Documentation Standards

Documentation is part of the delivery, not a task that happens after delivery. A change is not done until the documentation is done.

This is not about writing for its own sake. Every document type here serves a specific audience and a specific purpose. When documentation is current, the team can operate the system, understand past decisions, and hand off work without tribal knowledge. When documentation lags, the next person — including a future version of you — pays the cost.

AI-assisted work creates an additional obligation: the reasoning, the tradeoffs, and the AI contributions need to be captured, because AI leaves no natural artifact of its own thinking.

---

## Doc-as-You-Go Practice

Write documentation while the context is fresh — not after the PR is merged, not before the deadline, not "later."

**During design:** Write the design doc before writing code. Capturing the approach before implementation forces clarity and catches problems early.

**During implementation:** Update the runbook when you add a new operational concern. Add a CHANGELOG entry when the behavior changes. Write the ADR when a decision is made — not when you remember it a week later.

**During review:** The reviewer should see the documentation alongside the code diff. If the docs aren't there, the review is incomplete.

**After release:** Update the README if the system's interface or behavior changed. Close the documentation ticket alongside the engineering ticket.

The test: if someone unfamiliar with this change had to operate this system at 2am, could they do it using only the documentation in this repository?

---

## Document Types and When to Use Them

### Architecture Decision Record (ADR)

**Purpose:** Record why a significant technical decision was made, what alternatives were considered, and what tradeoffs were accepted.

**When to write one:** Any decision that would be costly to reverse, affects multiple systems or teams, or that a future engineer might question without context. If you catch yourself saying "we chose X because..." in a PR comment, write an ADR instead.

**Audience:** Future engineers, technical reviewers, anyone who inherits this codebase.

**Template:** [templates/architecture-decision-record.md](../templates/architecture-decision-record.md)

**Example:** [examples/sample-architecture-decision-record.md](../examples/sample-architecture-decision-record.md)

---

### Design Doc

**Purpose:** Describe the approach to a non-trivial piece of work before it is built. Covers context, requirements, proposed design, alternatives considered, open questions, and risks.

**When to write one:** Any change that touches 3+ files, introduces a new abstraction, changes a public interface, or involves meaningful tradeoffs. Not required for small bug fixes or minor config changes.

**Audience:** The reviewer, the approver, and anyone who needs to understand why the system works the way it does.

**Template:** [templates/design-doc.md](../templates/design-doc.md)

---

### Runbook

**Purpose:** Step-by-step operational procedures for a specific component or scenario. Runbooks answer: "What do I do when X happens at 2am?"

**When to write one:** When you add a new service, a new failure mode, a new alert, or a new manual intervention procedure.

**Audience:** On-call engineers, operators, anyone responding to an incident.

**Template:** [templates/runbook.md](../templates/runbook.md)

---

### CHANGELOG

**Purpose:** A human-readable record of what changed between versions, organized by release. Follows the [Keep a Changelog](https://keepachangelog.com/) format.

**When to update:** Every PR that changes user-facing behavior, fixes a bug, changes an API, or removes a feature.

**Audience:** Users of the system, stakeholders tracking what shipped, engineers reviewing release history.

**Format:** Entries under `Added`, `Changed`, `Fixed`, `Deprecated`, `Removed`, `Security`.

---

### README

**Purpose:** Orientation for someone encountering the repository or service for the first time. Covers what the thing is, how to set it up, how to run it, and where to find more detail.

**When to update:** When the project's interface, setup steps, or high-level behavior changes.

**Audience:** New engineers, contributors, anyone evaluating the project.

---

### AI Change Log

**Purpose:** Record what AI tools contributed to a change — which prompts were used, what was generated, what was modified by a human reviewer, and what was rejected.

**When to write one:** Any time AI-generated code is included in a PR. Not required for AI-assisted documentation or analysis where the human wrote the final output.

**Why it matters:** Reviewers need to know where to apply extra scrutiny. Auditors need traceability. The methodology requires it.

**Template:** [templates/ai-change-log.md](../templates/ai-change-log.md)

---

## Change Type → Required Documentation

| Change type | ADR | Design doc | Runbook update | CHANGELOG entry | README update | AI change log |
|---|---|---|---|---|---|---|
| New service or major feature | Required | Required | Required | Required | If user-facing | If AI contributed code |
| New API endpoint | If design decision | Required | Optional | Required | If public API | If AI contributed code |
| Significant refactor | If changes abstraction | Required | If operational impact | Required | If interface changes | If AI contributed code |
| Bug fix | No | No | If changes failure handling | Required | No | If AI contributed code |
| Config or env change | If significant | No | If operationally relevant | Required | If affects setup | No |
| Dependency update | If major/breaking | No | No | Required | No | No |
| Documentation only | No | No | No | No | If changes orientation | No |
| Security fix | No (avoid detail) | No | If changes response procedure | Required (brief) | No | If AI contributed code |

"Required" means the PR will not be approved without it. "Optional" means it is encouraged when it adds value.

---

## Documentation Quality Bar

Good documentation meets these criteria:

- **Specific.** Concrete examples, not vague descriptions. "The `/dispatch` endpoint accepts up to 100 recipient IDs per call" is better than "the endpoint supports batching."
- **Current.** Reflects the current state of the system, not the state when it was first written.
- **Findable.** Cross-linked from the places where someone would look for it.
- **Proportionate.** Effort matches the complexity and risk of the change. A three-line bug fix does not need a five-page design doc.
- **Honest about uncertainty.** If a decision was a judgment call with real tradeoffs, say so. Do not write documentation that sounds more certain than the underlying decision was.

AI-generated documentation needs the same review that AI-generated code gets. Check it for accuracy, specificity, and correctness before committing.

---

## Template Index

| Template | Use for |
|---|---|
| [templates/architecture-decision-record.md](../templates/architecture-decision-record.md) | Recording technical decisions |
| [templates/design-doc.md](../templates/design-doc.md) | Planning non-trivial changes |
| [templates/test-plan.md](../templates/test-plan.md) | Documenting test strategy per change |
| [templates/release-plan.md](../templates/release-plan.md) | Planning and executing a release |
| [templates/incident-postmortem.md](../templates/incident-postmortem.md) | Learning from production incidents |
| [templates/risk-assessment.md](../templates/risk-assessment.md) | Identifying and tracking delivery risks |
| [templates/ai-change-log.md](../templates/ai-change-log.md) | Recording AI contributions to a change |
| [templates/runbook.md](../templates/runbook.md) | Operational procedures |
| [templates/pull-request.md](../templates/pull-request.md) | Consistent PR descriptions |

---

## Related Documents

- [docs/ai-assisted-workflow.md](ai-assisted-workflow.md) — the broader workflow that documentation standards support
- [docs/human-in-the-loop.md](human-in-the-loop.md) — human review requirements, including documentation review
- [docs/delivery-lifecycle.md](delivery-lifecycle.md) — where documentation fits in the full delivery lifecycle
- [checklists/documentation.md](../checklists/documentation.md) — documentation checklist used during PR review
