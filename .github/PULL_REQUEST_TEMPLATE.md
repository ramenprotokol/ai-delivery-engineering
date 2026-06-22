## Summary

<!-- One or two sentences. What does this PR do and why does it exist? -->

## Type of change

- [ ] New content (doc, template, checklist, example)
- [ ] Update to existing content
- [ ] Bug fix (broken link, incorrect information, typo)
- [ ] Structural / repo hygiene (config, workflows, metadata)
- [ ] Other: <!-- describe -->

## What changed and why

<!--
Be specific. "Updated the release checklist" is weak. "Added a rollback verification step to the release checklist because the previous version had no explicit check that a rollback path existed before go/no-go" is strong.
-->

## How this was validated

<!--
Evidence that the change is correct. For docs and templates: did you cross-check against the existing examples? For workflows: did the CI run pass? For checklists: did you walk through a concrete scenario?
-->

- [ ] CI passed (markdownlint + link check)
- [ ] Manually reviewed for accuracy against the [docs/](../docs/) and [examples/](../examples/) directories
- [ ] Cross-links verified (relative paths resolve correctly)
- [ ] No placeholder text, TODOs, or lorem ipsum left in the output

## AI assistance

- [ ] AI-assisted (Claude Code, Cursor, or similar used during authoring or review)
- [ ] Human-reviewed (a human read and validated the content and reasoning)
- [ ] Human-approved (a human made the final call to merge)

<!-- If AI was used, briefly note what it helped with (drafting, cross-checking, refactoring). -->

## Risk and rollback

<!--
Most PRs to a docs repo are low risk. Flag anything that isn't:
- Changing a template structure that downstream workflows depend on
- Altering a checklist that has been cited in published content
- Updating a CI workflow
-->

**Risk level:** Low / Medium / High

**Rollback plan:** <!-- Revert this commit. Note any secondary effects. -->

## Checklist

Review the [pre-merge checklist](../checklists/pre-merge.md) before requesting a review.

- [ ] Pre-merge checklist complete
- [ ] PR title is clear and describes the change, not the work session
- [ ] Scope is focused — one logical change per PR
