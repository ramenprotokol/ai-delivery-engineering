# Contributing to ai-delivery-engineering

Thank you for your interest in contributing. This is a public methodology and documentation repository. Contributions that improve precision, add real engineering perspective, or extend the framework with coherent examples are welcome.

Read this file before opening a pull request.

---

## Scope of contributions

This repo documents how to plan, deliver, validate, and operate AI-assisted software systems. Contributions in scope:

- Corrections to factual or technical inaccuracies
- Improvements to clarity, structure, or completeness of existing docs
- New templates or checklists that fit the established framework
- New worked examples using the synthetic universe (Beacon, Atlas, Orchard — see below)
- Fixes to broken links, formatting issues, or CI failures

**Out of scope:**

- Content promoting specific vendors without engineering rationale
- Opinion pieces or blog-style posts
- Anything that references real people, real companies, or real private systems (see Privacy below)
- Changes that contradict the core positioning without a strong technical argument

If you are unsure whether your contribution fits, open a GitHub Issue before writing anything.

---

## Privacy and synthetic examples

This is a public repository. All examples must use the synthetic universe defined in the project:

| Synthetic name | What it represents |
|---|---|
| **Beacon** | A fictional transactional notification/delivery REST API |
| **Atlas** | A fictional internal developer platform and CLI |
| **Orchard** | A fictional task-management web app |

Refer to people by role only: "the delivery engineer", "a reviewer", "on-call", "the approver". Never use real names, email addresses, company names, customer data, revenue figures, or private system details. Pull requests that include personal or private information will be rejected without merge.

---

## Documentation style

Plain English. Short sentences. Specific and concrete — if a sentence could apply to any software project anywhere, rewrite it to apply to this one.

**Banned phrases:** leverage, synergy, paradigm, robust, scalable framework, holistic, move the needle, low-hanging fruit, game-changer, best-in-class.

**Prefer:**

- Active voice
- Present tense for current state, past tense for what happened in an example
- Real numbers and real conditions over vague generalizations
- Tables and fenced code blocks where they add clarity over prose
- Mermaid diagrams for flows and lifecycles (see existing docs for examples)

Every file must have exactly one H1. Cross-links between files use relative paths from the file being written, not absolute paths.

---

## AI usage in contributions

You may use AI tools (Claude Code, Cursor, GitHub Copilot, or similar) to draft or edit content. If you do:

- You are responsible for reviewing every line before submitting
- You own the accuracy and technical correctness of what you submit
- Shallow AI boilerplate, placeholder text, "TODO" items, or lorem ipsum will cause the PR to be rejected
- Do not include AI-generated content that you have not read, understood, and verified

The project policy is "AI-assisted. Human-reviewed. Human-approved." That standard applies to contributors as much as it applies to the work documented here.

---

## Branch and PR workflow

1. Fork the repository.
2. Create a branch from `main` with a short, descriptive name:

   ```text
   docs/add-glossary-term-drift
   fix/broken-link-validation-framework
   feat/checklist-canary-deployment
   ```

3. Make your changes. Keep commits focused — one logical change per commit.
4. Run local checks (see below) before pushing.
5. Open a pull request against `main`. Use the pull request template at [`.github/PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md).
6. A maintainer will review. Expect a response within a few business days.

Do not open PRs that touch unrelated files. Do not force-push to a branch that already has an open PR without noting it in the PR comments.

---

## Commit message convention

Follow this format:

```text
<type>(<scope>): <short description>

[optional body — what changed and why, if not obvious]
```

**Types:**

| Type | Use for |
|---|---|
| `docs` | Documentation changes (most contributions will be this) |
| `feat` | New template, checklist, or example |
| `fix` | Correction to existing content or broken link |
| `ci` | Changes to GitHub Actions workflows |
| `chore` | Dependency updates, repo hygiene |

**Examples:**

```text
docs(glossary): add definition for 'model drift'
fix(checklists): correct broken link to validation-framework
feat(templates): add canary deployment runbook template
```

Keep the subject line under 72 characters. Use the body for context when the change is non-obvious.

---

## Local checks before opening a PR

The CI pipeline (see [`.github/workflows/ci.yml`](.github/workflows/ci.yml)) runs markdown linting and link checking on every PR. Run these locally to catch failures before pushing.

**Markdown lint:**

```bash
npx markdownlint-cli2 "**/*.md"
```

The command auto-discovers `.markdownlint-cli2.jsonc` at the repo root — no extra flags needed. See that file for the full rule set.

**Link check:**

```bash
lychee --config lychee.toml .
```

Or run the full CI check locally with:

```bash
act pull_request
```

(Requires [act](https://github.com/nektos/act) and Docker.)

Fix all lint errors and broken links before opening your PR. PRs that fail CI will not be merged until the checks pass.

---

## Review and approval

All pull requests require at least one maintainer approval before merge. Maintainers may:

- Request changes to style, scope, or technical accuracy
- Ask for a synthetic example to replace a vague description
- Reject contributions that fall outside scope or violate the privacy policy

Reviews focus on technical credibility, consistency with the existing framework, and clarity. They are not personal.

---

## Questions

Open a GitHub Issue on [ramenprotokol/ai-delivery-engineering](https://github.com/ramenprotokol/ai-delivery-engineering). Do not contact maintainers via personal channels.
