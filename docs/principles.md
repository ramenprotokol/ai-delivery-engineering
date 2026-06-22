# Principles

These are the engineering values that drive every practice in this repository. They are not aspirational slogans — each one has a concrete consequence for how work is planned, built, validated, and shipped.

---

## 1. AI Accelerates. Humans Are Accountable

AI tools generate code, surface options, and reduce mechanical work. They do not make decisions, own outcomes, or carry accountability. The engineer who commits a change owns it. The reviewer who approves it owns the approval. Neither can delegate accountability to the model that helped write it.

**In practice:** When a bug ships in AI-generated code, the postmortem traces back to the human who reviewed and approved it — not the tool that wrote it.

---

## 2. Validation Over Trust

AI output is a draft, not a deliverable. Trusting that AI-generated code is correct because it looks right is not engineering — it is wishful thinking. Every output is validated: tested, linted, read, understood, and checked against acceptance criteria before it moves forward.

**In practice:** A passing CI run is necessary but not sufficient. The engineer also checks that the tests actually exercise the right behavior, not just that they pass.

---

## 3. Small, Reversible Changes

Large changes increase the blast radius of errors and make rollback expensive or impossible. Work is broken into small, independently deployable units. Each unit can be reversed without unrolling everything that came after it.

**In practice:** Before opening a pull request, the delivery engineer asks: "If this breaks in production tonight, can I roll it back in under ten minutes?" If the answer is no, the change is too large.

---

## 4. Evidence Before "Done"

"Done" is not a feeling — it is a state backed by evidence. Evidence includes passing tests, a successful deployment to a staging environment, a completed checklist, and approval from a second human. A change that feels done but lacks evidence is not done.

**In practice:** The [release readiness checklist](../checklists/release-readiness.md) is completed and attached to every pull request before the merge review begins.

---

## 5. Automate the Checklist, Not the Judgment

Repetitive verification steps belong in CI: formatting checks, lint, type checks, unit tests, dependency audits. But the decision of whether a change is safe, complete, and worth shipping belongs to a human. Automating a gate does not mean removing the human from it.

**In practice:** CI enforces that tests pass and the build is clean. A human still reviews the diff and confirms the change does what it claims to do.

---

## 6. Make AI Involvement Auditable

When AI generates a significant portion of a change — code, documentation, a design decision — that involvement is noted. Not because AI work is suspect, but because reviewers and future maintainers deserve to know the provenance of what they are reading. Transparency makes the review more honest and the codebase more trustworthy.

**In practice:** Pull requests and [AI change logs](../templates/ai-change-log.md) record which parts of a change were AI-assisted and what human review was applied.

---

## 7. Optimize for the Reviewer

Code is written once and read many times. A change that is easy to write but hard to review is a net cost to the team. Structure diffs to tell a clear story. Write commit messages that explain why, not just what. Link to the issue, the ADR, or the design doc when context matters.

**In practice:** Before opening a pull request, the author reads their own diff as if they are a reviewer seeing it for the first time. If anything is unclear, it is clarified before review begins.

---

## 8. Reversibility and Rollback First

Every change that reaches production needs a rollback path defined before deployment, not after an incident starts. The rollback plan is part of the release plan — not an afterthought.

**In practice:** The [release plan template](../templates/release-plan.md) includes a required rollback section. A release plan without a rollback section is incomplete and cannot be approved.

---

## 9. Assume the System Will Fail

Production systems fail in unexpected ways. Observability is not optional — it is the mechanism by which engineers learn what is actually happening. Alerts, dashboards, structured logs, and error tracking are configured before a change goes live, not after an incident reveals they were missing.

**In practice:** The [production readiness checklist](../checklists/production-readiness.md) requires that monitoring and alerting for the changed surface are verified before deployment.

---

## 10. Judgment Is the Job

Speed is valuable. Shipping is valuable. But the judgment to know when something is not ready — when a test is not actually testing the right thing, when a design decision has a hidden flaw, when a change needs to be smaller — is the core of the work. AI makes the mechanical parts faster. It does not replace the judgment that makes the work good.

**In practice:** When an AI tool confidently produces an answer, the engineer asks: "What would have to be true for this to be wrong?" That question is the job.
