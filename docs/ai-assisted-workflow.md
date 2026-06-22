# AI-Assisted Workflow

This document explains concretely how AI tools are integrated at each stage of the [delivery lifecycle](delivery-lifecycle.md). It covers patterns that work, guardrails, what AI is and isn't good for, and how the workflow catches AI failure modes before they reach production.

The through-line: AI accelerates execution. Humans own architecture, security trade-offs, release decisions, and accountability.

---

## Patterns That Work

### Scoped Prompts

Vague prompts produce vague output. The delivery engineer writes a prompt that includes:

- What the code/doc must do
- Constraints (language, library version, existing interface it must match)
- What the output format should be
- What it must NOT do

**Example prompt (Beacon queue handler):**

```text
Write a Go function that calls queue.PutMessage(ctx, msg). If PutMessage returns an error,
return HTTP 500 with a JSON error body. If it succeeds, return HTTP 200 with {"queued": true}.
Use the existing `respondJSON` helper. Do not add logging — that's handled by middleware.
Timeout for PutMessage is passed in as a parameter, not hardcoded.
```

A scoped prompt gives the AI enough context to produce something useful and enough constraints to keep it from inventing its own architecture.

### Generate-Then-Validate

AI output is a first draft, not a final answer. The delivery engineer:

1. Generates the output (code, test plan, ADR draft, etc.)
2. Reads it — not skims it, reads it
3. Checks it against the actual source of truth (API docs, specs, existing codebase)
4. Edits where the AI was wrong, incomplete, or made up an interface
5. Logs significant AI-generated blocks in the [AI Change Log](../templates/ai-change-log.md)

This is not a bureaucratic step. It is how hallucinated API signatures, incorrect library versions, and plausible-but-wrong logic get caught before they merge.

### AI for Breadth, Human for Judgment

AI is fast at generating many options, many test cases, many edge cases. Humans are better at evaluating which option is right given constraints that aren't fully stated in the prompt.

Use AI to expand the possibility space. Use human judgment to collapse it.

**Example:** When designing the Beacon queue write confirmation, the AI suggested five implementation approaches. The delivery engineer used that list as a starting point, evaluated each against the actual latency budget and failure mode requirements, and documented the choice in the ADR. The AI didn't make the decision. It made sure the decision was made with more options on the table.

### Paired Review of AI Output

For significant AI-generated blocks, the delivery engineer reviews the AI output against an independent source before trusting it. For code: verify the method signature against the SDK docs. For security claims: check the CVE database or library changelog. For architecture claims: trace the reasoning, don't accept the conclusion.

If a second engineer is available, they review AI-generated code with fresh eyes. The reviewer explicitly uses the [AI code review checklist](../checklists/ai-code-review.md), which includes prompts specific to AI failure modes.

---

## AI Integration by Lifecycle Stage

| Stage | What AI Does | What AI Does Not Do |
|---|---|---|
| Intake / Scope | Drafts problem statements, surfaces related issues, suggests acceptance criteria | Confirms the problem is real; prioritizes work |
| Design | Generates ADR/design doc skeletons, suggests alternatives, identifies edge cases | Chooses the architecture; owns the trade-off decision |
| Build | Generates implementation code, suggests tests, writes migration scripts | Owns the diff; verifies its own output against actual docs |
| Validate | Suggests additional test cases, reviews test plan for gaps, summarizes test output | Decides whether acceptance criteria are met |
| Review | Assists reviewer in surfacing issues, generates review summaries | Approves the pull request; replaces the reviewer's judgment |
| Release | Drafts release plans and stakeholder communications, suggests rollback triggers | Makes the go/no-go decision |
| Operate / Learn | Summarizes log patterns, drafts postmortem timelines, suggests process improvements | Owns incident response; signs off on postmortems |

---

## What AI Is Good For vs. What Stays a Human Decision

| AI is good for | Human decision, not AI |
|---|---|
| Generating boilerplate and repetitive code patterns | Architecture: which approach to use and why |
| Expanding the set of options or edge cases to consider | Security trade-offs: what risk to accept |
| Producing a first draft of documentation or a design doc | Release go/no-go: whether to ship now |
| Translating a rough description into structured text | Incident response: what to do under pressure |
| Summarizing large diffs or log output | Accountability: who owns this change |
| Finding syntax errors and obvious logic bugs | Deciding what "correct" means for this system |
| Suggesting test cases given a function signature | Confirming acceptance criteria are actually met |
| Generating a runbook from a described procedure | Validating that the runbook is operationally sound |

This split is not about AI capability. It is about accountability. A named human is accountable for every decision in the table's right column. AI cannot be accountable. Humans can.

---

## The AI Change Log

Every significant AI-generated block is logged in the [AI Change Log](../templates/ai-change-log.md). "Significant" means: any code that will be merged, any design decision that came from AI suggestion, or any test plan section generated by AI.

The log records:

- What was generated (file, function, or section)
- Which AI tool was used
- The prompt (or a summary if the prompt was long)
- What the human reviewer changed or verified
- Whether the AI output was used as-is, modified, or discarded

This is an audit trail, not a compliance ritual. It answers the question "how much of this was AI and how was it validated?" when that question comes up in a code review, an incident postmortem, or a security review.

See [templates/ai-change-log.md](../templates/ai-change-log.md) for the template and field definitions.

---

## The AI Code Review Checklist

Reviewing AI-generated code requires different attention than reviewing human-written code. AI code tends to:

- Use the correct structure but the wrong method signature
- Handle the happy path well and miss error cases
- Add plausible-looking but incorrect comments
- Use deprecated APIs (especially if the model's training predates a major library version)
- Introduce subtle logic errors in conditionals and boundary conditions

The [AI code review checklist](../checklists/ai-code-review.md) is specifically designed for this. It prompts the reviewer to check method signatures against actual docs, verify error handling explicitly, and flag any comment that asserts a behavior the reviewer hasn't independently confirmed.

---

## How the Workflow Catches AI Failure Modes

### Hallucinated APIs

**Failure mode:** AI generates code that calls a method that doesn't exist, or exists with a different signature.

**How it's caught:** The delivery engineer verifies every external method call against the actual SDK or API documentation during Stage 3 (Build). The AI change log records this verification. The reviewer re-checks flagged methods during Stage 5 (Review).

**Beacon example:** Claude Code generated a call to `queue.EnqueueWithAck(ctx, msg, opts)`. That method does not exist in the queue SDK. The actual method is `queue.PutMessage(ctx, msg)` with acknowledgement handled by the return value. The delivery engineer caught this during the generate-then-validate step, corrected it, and logged the correction.

### Confident-But-Wrong Logic

**Failure mode:** AI generates code that looks correct but has a logic error — an off-by-one, an inverted condition, a missing nil check — that passes a casual read.

**How it's caught:** Tests written against acceptance criteria (not just against the code) catch behavior errors. The AI code review checklist prompts reviewers to trace error paths explicitly. Any AI-generated conditional with a non-obvious boundary is flagged for a second look.

### Outdated Patterns

**Failure mode:** AI generates code using a pattern that was correct for an older version of a library or language spec.

**How it's caught:** The delivery engineer checks the library version in the project's dependency manifest against what the AI assumed. If there's a version gap, the generated code is checked against the current changelog. This is explicitly on the AI code review checklist.

### Scope Creep in Generation

**Failure mode:** The AI generates more than was asked for — adds logging, changes error handling, refactors adjacent code — and the engineer merges it without noticing.

**How it's caught:** Scoped prompts reduce this. The delivery engineer reads the full diff before opening a PR — not just the function that was requested. The PR review process catches changes to files that weren't expected to change.

---

## Guardrails Summary

1. AI output is never merged without a human reading the complete diff
2. Every significant AI-generated block is logged in the AI change log
3. Method signatures and library calls are verified against actual documentation, not the AI's description of it
4. The reviewer uses the AI code review checklist — not a generic review checklist
5. Security-relevant code (auth, input validation, encryption, permissions) gets an independent human review regardless of source
6. The go/no-go decision for any release is made by a named human

---

## Related

- [Delivery Lifecycle](delivery-lifecycle.md) — the full lifecycle this workflow operates within
- [Human-in-the-Loop](human-in-the-loop.md) — the accountability and review model
- [AI Change Log template](../templates/ai-change-log.md) — the audit trail for AI-generated work
- [AI Code Review Checklist](../checklists/ai-code-review.md) — review prompts specific to AI-generated code
- [Validation Framework](validation-framework.md) — how validation works
