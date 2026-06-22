# Human-in-the-Loop

AI writes code, drafts designs, suggests tests, and summarizes incidents. A named human is still accountable for every decision and every change that reaches production.

This document defines the accountability and review model: who reviews what and when, what the three labels mean (AI-assisted, human-reviewed, human-approved), how review gates work, and what the audit trail looks like.

---

## The Three Labels

These labels appear on pull requests, design docs, and release records. They are not marketing. They are a precise description of how a piece of work was produced and verified.

**AI-assisted** — AI tools contributed to producing this artifact. The delivery engineer directed the AI, reviewed the output, and owns the result. This label is informational: it tells reviewers to apply the [AI code review checklist](../checklists/ai-code-review.md).

**Human-reviewed** — A human (not the author) has read the artifact, checked it against the design and acceptance criteria, and is satisfied that it does what it claims. This is a statement of understanding, not just approval. The reviewer's name is recorded.

**Human-approved** — A named human has made a decision: this change is safe to ship, this design is the right approach, this postmortem is accurate. Approval carries accountability. The approver's name and timestamp are in the record.

A change can be AI-assisted and human-reviewed and human-approved — those labels stack. A change cannot be human-approved without also being human-reviewed.

---

## Review Gates

Each gate in the [delivery lifecycle](delivery-lifecycle.md) has a defined reviewer, a defined artifact, and a defined question the reviewer must answer.

| Gate | Stage | Reviewer | Artifact | Question answered |
|---|---|---|---|---|
| Scope confirmation | Intake | Delivery engineer | Tracking issue | Is the problem statement correct and are acceptance criteria testable? |
| Design approval | Design | Delivery engineer + (second reviewer for significant changes) | ADR / Design Doc | Is this the right approach given constraints, risks, and operational reality? |
| PR readiness | Build | Author (self-gate) | Pull request diff + AI change log | Have I read the full diff, verified AI-generated calls, and confirmed tests pass? |
| Validation sign-off | Validate | Delivery engineer | Completed test plan | Are all acceptance criteria demonstrably met? |
| Code review approval | Review | Peer reviewer (not the author) | Pull request | Is this safe to merge? |
| Release go/no-go | Release | Approver (delivery engineer or lead) | Release plan | Is this safe to ship now? |
| Postmortem sign-off | Operate / Learn | Reviewer (second engineer or lead) | Postmortem | Is this timeline accurate and do the action items address the root cause? |

No gate is skipped, even for small changes. The size of a change does not reduce the need for human review — it reduces the time the review takes.

---

## Who Reviews What

**Delivery engineer** — owns the change end to end. Reviews their own work before every handoff (self-gate at PR readiness, validation sign-off). Accountable for the correctness of AI change log entries. Cannot approve their own pull request.

**Peer reviewer** — a second engineer who has not been involved in building the change. Reads the full diff. Uses the [AI code review checklist](../checklists/ai-code-review.md) when AI-assisted code is present. Can request changes; cannot skip the review because the diff looks small or the author seems confident.

**Approver** — makes the release go/no-go decision. This may be the delivery engineer for routine changes, a lead for significant ones. The approver confirms the release plan is complete, rollback is validated, and post-deploy monitoring is in place. The approver's name is in the deploy record.

**On-call engineer** — owns incident response in Stage 7. Does not delegate triage decisions to AI. May use AI to summarize logs or draft postmortem timelines, but owns every action taken during the incident.

---

## RACI for a Synthetic Change

**Change:** Add idempotency key support to Beacon's `/notify` endpoint so duplicate requests within a 5-minute window are deduplicated rather than double-queued.

| Activity | Delivery Engineer | Peer Reviewer | Approver | On-Call |
|---|---|---|---|---|
| Write problem statement and acceptance criteria | **Responsible** | Consulted | — | — |
| Approve scope and criteria | **Accountable** | — | — | — |
| Draft design doc (AI-assisted) | **Responsible** | — | — | — |
| Approve design | Responsible | **Accountable** | — | — |
| Write implementation (AI-assisted) | **Responsible** | — | — | — |
| Log AI-generated blocks in AI change log | **Accountable** | — | — | — |
| Write and run tests | **Responsible** | — | — | — |
| Validate acceptance criteria are met | **Accountable** | — | — | — |
| Review pull request | Consulted | **Accountable** | — | — |
| Approve pull request | — | **Accountable** | — | — |
| Make release go/no-go decision | Consulted | — | **Accountable** | — |
| Execute deploy | **Responsible** | — | Accountable | — |
| Monitor post-deploy health checks | **Responsible** | — | Informed | — |
| Respond to any post-deploy incident | Informed | — | Informed | **Accountable** |
| Sign off on postmortem | Consulted | — | — | Responsible, **Accountable** |

One person is Accountable for each row. Accountability does not spread across roles. If something goes wrong, there is a specific human who owned that decision.

---

## The Audit Trail

Two artifacts together form the audit trail for every AI-assisted change.

### AI Change Log

The [AI Change Log](../templates/ai-change-log.md) records, for each merged change:

- Which AI tool was used
- What was generated (file, function, section)
- The prompt or a summary of it
- What the human reviewer changed before accepting the output
- Whether the output was used as-is, modified, or discarded

This answers the question "what role did AI play in this change?" during code review, security review, or incident investigation. The delivery engineer fills this log during Build (Stage 3); it is reviewed during code review (Stage 5).

### Pull Request Record

The PR record captures:

- The author (who built it)
- The reviewer(s) (who approved it)
- The approver (who decided to ship it)
- The timestamp of each action
- The commit SHAs deployed

Together, the AI change log and PR record mean that for any line in production, there is a chain: what AI generated it, who reviewed it, who approved it, and when. This chain must be unbroken. A change that lacks an AI change log entry for significant AI-generated blocks, or lacks a named reviewer, is not complete — even if the code is correct.

---

## Why a Named Human Must Be Accountable

AI tools generate plausible output. Plausible is not the same as correct. A tool that produces confident, well-formatted, syntactically valid code that silently does the wrong thing is a real failure mode — not a theoretical one.

Human accountability is not a compliance checkbox. It is the mechanism that ensures someone has actually understood what will run in production. The delivery engineer who reads the diff, verifies the AI-generated method signatures, and signs the PR has done something the AI cannot do: they have applied judgment to the specific context of this system, this codebase, and this moment.

When something goes wrong — and eventually something always goes wrong — the question is not "what did the AI do?" but "who decided this was safe to ship?" That question needs a real answer.

---

## Related

- [Delivery Lifecycle](delivery-lifecycle.md) — the stages where each gate lives
- [AI-Assisted Workflow](ai-assisted-workflow.md) — how AI is used within each stage
- [AI Change Log template](../templates/ai-change-log.md) — the primary audit artifact
- [AI Code Review Checklist](../checklists/ai-code-review.md) — review checklist for AI-generated code
- [Pull Request template](../.github/PULL_REQUEST_TEMPLATE.md) — captures review and approval
