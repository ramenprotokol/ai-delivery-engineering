# Changelog

All notable changes to this project will be documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

---

## [0.1.0] - 2026-06-21

### Added

**Methodology docs** (`docs/`)

- `methodology-overview.md` — top-level map of the delivery methodology and how the pieces fit together
- `principles.md` — the ten core principles governing AI-assisted delivery (validation-first, human accountability, etc.)
- `glossary.md` — shared vocabulary for the repo (AI change log, delivery envelope, gate, human-in-the-loop, etc.)
- `delivery-lifecycle.md` — end-to-end lifecycle from intake to production: phases, gates, handoffs
- `ai-assisted-workflow.md` — where AI fits in the workflow, what it accelerates, and where humans stay in the loop
- `human-in-the-loop.md` — decision model for when to require human review vs. allow AI autonomy
- `validation-framework.md` — layered approach to validating AI-generated artifacts (static, dynamic, human review)
- `testing-strategy.md` — testing philosophy and coverage expectations across unit, integration, contract, and E2E layers
- `release-readiness.md` — definition of "ready to ship": criteria, gate owners, and sign-off requirements
- `risk-management.md` — risk identification, classification, and mitigation process for AI-assisted changes
- `security-practices.md` — security considerations specific to AI-assisted development (prompt injection, data exposure, supply chain)
- `documentation-standards.md` — what to document, how to document it, and what counts as done

**Templates** (`templates/`)

- `architecture-decision-record.md` — structured ADR with context, options considered, decision, and consequences
- `design-doc.md` — lightweight design document for features and system changes
- `test-plan.md` — test plan covering scope, approach, environment, cases, and acceptance criteria
- `release-plan.md` — release plan with checklist, rollback trigger, and communication steps
- `incident-postmortem.md` — blameless postmortem with timeline, root cause, and action items
- `risk-assessment.md` — per-change risk scoring with likelihood, impact, and mitigation columns
- `ai-change-log.md` — structured log of AI-generated and AI-assisted changes with human review status
- `runbook.md` — operational runbook for a service or component (health checks, on-call steps, escalation)
- `pull-request.md` — PR description template with change summary, test evidence, and review checklist
- `README.md` — index of all templates with a one-line description each

**Checklists** (`checklists/`)

- `pre-merge.md` — gates every pull request must pass before merge (tests, review, docs, security scan)
- `release-readiness.md` — gates a release must pass before production deployment
- `ai-code-review.md` — specific questions to ask when reviewing AI-generated code
- `security-review.md` — security review checklist covering auth, input validation, secrets, and dependencies
- `production-readiness.md` — service-level readiness (observability, alerting, runbook, rollback plan)
- `documentation.md` — checklist confirming docs are complete, accurate, and linked
- `testing.md` — checklist confirming test coverage is adequate and results are clean
- `README.md` — index of all checklists with use-case guidance

**Examples** (`examples/`)

- `feature-delivery-walkthrough.md` — complete worked example: Beacon rate-limit feature from intake to production using the full methodology
- `bugfix-walkthrough.md` — worked example: diagnosing and shipping a fix for a silent data-loss bug in Beacon's event pipeline
- `ai-assisted-refactor.md` — worked example: AI-assisted refactor of Atlas CLI authentication module with full human validation trail
- `sample-architecture-decision-record.md` — filled ADR: choosing idempotency strategy for Beacon's notification delivery
- `sample-test-plan.md` — filled test plan for Orchard task-assignment feature
- `sample-release-plan.md` — filled release plan for Beacon v2.3.0
- `sample-postmortem.md` — filled postmortem for a Beacon delivery-rate regression
- `README.md` — index of all examples with context on how to read them

**GitHub config** (`.github/`)

- `PULL_REQUEST_TEMPLATE.md` — default PR template applied to every pull request in the repo
- `ISSUE_TEMPLATE/bug_report.md` — structured bug report template
- `ISSUE_TEMPLATE/feature_request.md` — structured feature request template
- `ISSUE_TEMPLATE/config.yml` — disables blank issues; directs users to the correct template
- `workflows/ci.yml` — CI workflow: lint markdown, validate links, check YAML and CFF syntax on every push and PR

**Root meta**

- `README.md` — repository overview, positioning, quick-start, and navigation index
- `LICENSE` — MIT License, copyright 2026 Ramen Protocol
- `.gitignore` — OS, editor, env/secrets, logs, Node, Python, and coverage exclusions
- `.editorconfig` — UTF-8, LF line endings, 2-space indent for md/yml/json, trailing-whitespace rules
- `CITATION.cff` — Citation File Format 1.2.0 metadata for academic and professional reference
- `CONTRIBUTING.md` — contribution guide: how to open issues, propose changes, and submit pull requests
- `CODE_OF_CONDUCT.md` — Contributor Covenant 2.1 code of conduct
- `SECURITY.md` — responsible disclosure policy and contact channel

---

[Unreleased]: https://github.com/ramenprotokol/ai-delivery-engineering/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/ramenprotokol/ai-delivery-engineering/releases/tag/v0.1.0
