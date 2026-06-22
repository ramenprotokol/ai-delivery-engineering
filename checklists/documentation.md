# Documentation Checklist

Use this checklist before merging any change that introduces new behavior, modifies existing behavior, adds a new API surface, introduces a new operational dependency, or fixes a bug that was reported by a user. Documentation is part of the change — a feature is not done until it can be found, understood, and operated by someone who was not on the team that built it.

Related files: [pre-merge checklist](pre-merge.md) · [documentation standards doc](../docs/documentation-standards.md) · [AI change log template](../templates/ai-change-log.md) · [ADR template](../templates/architecture-decision-record.md) · [design doc template](../templates/design-doc.md) · [runbook template](../templates/runbook.md)

---

## 1. README and Usage Docs

- [ ] The service or module README reflects the current behavior after this change — outdated sections have been updated, not left as-is
- [ ] Any new configuration options, environment variables, or feature flags are documented with their type, default value, and effect
- [ ] Setup and quickstart instructions still work after this change — if they break, they are fixed before merge
- [ ] Examples in the README use realistic, synthetic data — no placeholder strings like `TODO`, `<insert value>`, or `example.com` unless those are genuinely valid defaults

## 2. Architecture Decision Records (ADRs)

- [ ] If this change involves a non-obvious technical decision (technology choice, schema design, API contract, trade-off between approaches), an ADR has been written using [templates/architecture-decision-record.md](../templates/architecture-decision-record.md)
- [ ] The ADR records what was decided, why, what alternatives were considered, and what the consequences are — not just what was built
- [ ] The ADR is linked from the PR description so reviewers can understand the reasoning without reading the full code diff

## 3. Design Document

- [ ] If this change is non-trivial (introduces a new component, changes a significant interface, or touches more than three independent systems), a design doc exists using [templates/design-doc.md](../templates/design-doc.md)
- [ ] The design doc was written before implementation began and reviewed before code was written — not retrofitted after the fact
- [ ] The design doc reflects the final implementation, not the original proposal (differences are noted in a "Changes from original design" section if they are significant)

## 4. Runbook for New Operations

- [ ] If this change introduces a new operational concern — a new background job, a new external dependency, a new failure mode, a new admin procedure — a runbook section exists for it using [templates/runbook.md](../templates/runbook.md)
- [ ] The runbook covers: how to verify the operation is running correctly, what to do when it fails, and how to recover
- [ ] The runbook has been reviewed by someone who will be on-call for it — not just the author

## 5. Changelog

- [ ] A changelog entry has been added to `CHANGELOG.md` describing this change in plain English from the perspective of a consumer of the system
- [ ] The entry is under the correct version heading (or "Unreleased" if the version has not been cut yet)
- [ ] The entry notes whether this is a new feature, a behavioral change, a deprecation, or a bug fix — the category is explicit
- [ ] Breaking changes are marked prominently and include migration instructions

## 6. API Documentation

- [ ] New API endpoints are documented: path, method, authentication requirements, request schema, response schema, and example request/response
- [ ] Changed endpoint behavior is reflected in the API docs — any fields added, removed, or modified are noted
- [ ] Deprecated fields or endpoints are marked with a deprecation notice and a migration path
- [ ] OpenAPI/Swagger spec or equivalent contract file is updated and validated if one exists for this service

## 7. AI Change Log

- [ ] If AI tools (Claude Code, Cursor, Copilot, or similar) contributed substantially to this change, the [AI change log](../templates/ai-change-log.md) has been updated
- [ ] The AI change log entry records: what the AI generated, what was modified by human review, what was rejected and why, and the final human approval
- [ ] No AI-generated content was merged without human review — the log reflects actual review, not a rubber stamp

## 8. Inline Code Comments

- [ ] Non-obvious logic is explained with inline comments that describe intent ("why"), not just restate the code ("what")
- [ ] Any workarounds, known limitations, or technical debt are marked with a comment that includes context and, where applicable, a link to a ticket
- [ ] Public functions and exported types have documentation comments explaining their contract — parameters, return values, and error conditions
- [ ] Comments are accurate — outdated comments that no longer reflect the code have been updated or removed

---

## Sign-off

| Role | Name / Handle | Date |
|---|---|---|
| Author | | |
| Reviewer | | |

> Documentation reviewed here is part of the deliverable. A change that ships without accurate documentation transfers the cost of understanding to the next engineer, on-call responder, or user — usually at the worst possible time.
