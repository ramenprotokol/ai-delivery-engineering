# Templates

This directory contains reusable document templates for AI-assisted delivery work. Each template covers a specific artifact in the delivery lifecycle — from initial design through release and post-incident review.

## How to Use a Template

1. Copy the template file into your project repository (or a shared docs directory).
2. Replace every HTML comment (`<!-- ... -->`) with real content, and remove the `> **Example (synthetic):**` blockquotes once you have filled in your own values. Do not leave hints or example blocks in the final document.
3. Fill every field. If a section does not apply, write "N/A — [reason]" rather than deleting it. Empty sections hide gaps; explained gaps build trust.
4. Keep the completed document alongside the change that produced it. An ADR belongs next to the code it governs; a release plan belongs in the same PR as the deployment.
5. Record AI assistance honestly. Note which sections AI drafted and who reviewed and approved each one.

## Template Index

| Template | File | Purpose |
|---|---|---|
| Architecture Decision Record | [architecture-decision-record.md](architecture-decision-record.md) | Capture a design decision, the options considered, the outcome, and the rationale — so future engineers understand why, not just what. |
| Design Doc | [design-doc.md](design-doc.md) | Describe a proposed feature or system change before writing code — problem, approach, alternatives, risks, and approvals. |
| Test Plan | [test-plan.md](test-plan.md) | Define what will be tested, how, and what pass/fail looks like before a change ships. |
| Release Plan | [release-plan.md](release-plan.md) | Coordinate a production release — steps, rollout strategy, monitoring, rollback, and go/no-go sign-off. |
| Incident Postmortem | [incident-postmortem.md](incident-postmortem.md) | Document what happened, the timeline, root causes, and concrete follow-up actions after a production incident. |
| Risk Assessment | [risk-assessment.md](risk-assessment.md) | Identify and quantify risks for a change or project so they can be tracked and mitigated before they become incidents. |
| AI Change Log | [ai-change-log.md](ai-change-log.md) | Record every AI-generated or AI-assisted change — what was generated, what was reviewed, what was changed, and who approved it. |
| Runbook | [runbook.md](runbook.md) | Step-by-step operational guide for a specific system or task — written to be followed under pressure by someone who did not build it. |
| Pull Request | [pull-request.md](pull-request.md) | Standard PR description structure — summary, test evidence, reviewer checklist, and deployment notes. |

## Related Resources

- [Checklists](../checklists/README.md) — gate-based checklists that work alongside these templates
- [Examples](../examples/README.md) — completed examples of selected templates against the Beacon synthetic system
- [Docs: Delivery Lifecycle](../docs/delivery-lifecycle.md) — where each template fits in the end-to-end flow
- [Docs: Documentation Standards](../docs/documentation-standards.md) — conventions for formatting, naming, and linking
