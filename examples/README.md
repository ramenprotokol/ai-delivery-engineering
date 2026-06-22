# Examples

This directory contains worked demonstrations of the delivery methodology described in [`docs/`](../docs/README.md). Every example is synthetic. No real customer work, business data, or personal information appears anywhere in this repository.

## Fictional systems used throughout

| System | What it is |
|--------|-----------|
| **Beacon** | A fictional transactional notification and delivery REST API. The main running example for features, bugs, and incidents. |
| **Atlas** | A fictional internal developer platform and CLI. Used in tooling, config, and platform-level examples. |
| **Orchard** | A fictional task-management web app. Reserved for future product-facing UI examples; not yet used in the walkthroughs below. |

People are referred to by role only — "the delivery engineer", "a reviewer", "on-call", "the approver" — never by name.

## What these examples are

Each file is a worked demonstration of the methodology in action. They show:

- the reasoning a delivery engineer applies at each lifecycle stage
- where AI assisted and what it produced
- what the human reviewed, decided, and approved
- which artifacts were created (linked to the matching templates and checklists)

They are not transcripts of real projects. They are concrete illustrations of how the methodology described in this repo is applied to real-class problems.

## Examples

| File | What it demonstrates |
|------|---------------------|
| [feature-delivery-walkthrough.md](feature-delivery-walkthrough.md) | End-to-end feature delivery: adding rate limiting to Beacon's send API. Every lifecycle stage, every artifact, every AI-assist and human decision. |
| [bugfix-walkthrough.md](bugfix-walkthrough.md) | Systematic bugfix on Beacon: notifications duplicating under retry. Reproduce, root-cause, fix, regression test, postmortem note. |
| [ai-assisted-refactor.md](ai-assisted-refactor.md) | Safe AI-assisted refactor of Atlas's config loader. Characterization tests first, small reversible steps, human validates behavior unchanged. |
| [sample-architecture-decision-record.md](sample-architecture-decision-record.md) | A completed ADR using the team template. |
| [sample-test-plan.md](sample-test-plan.md) | A completed test plan for the Beacon rate-limiting feature. |
| [sample-release-plan.md](sample-release-plan.md) | A completed release plan for a Beacon minor release. |
| [sample-postmortem.md](sample-postmortem.md) | A completed incident postmortem from a Beacon production event. |

## How to read these

Start with [feature-delivery-walkthrough.md](feature-delivery-walkthrough.md) for a full end-to-end picture. The bugfix and refactor walkthroughs are self-contained — read them in any order. The sample documents are stand-alone completed artifacts that reference the matching templates in [`templates/`](../templates/README.md).
