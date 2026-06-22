# Glossary

Alphabetized definitions for every term used across this repository. Where a term is covered in depth by another document, a link is provided.

---

**ADR (Architecture Decision Record)**
A short document that records a significant architecture or design decision, the context that drove it, the options considered, and the reasoning behind the choice. See the [ADR template](../templates/architecture-decision-record.md) and [sample ADR](../examples/sample-architecture-decision-record.md).

**AI-Assisted**
Work where an AI tool (Claude Code, Cursor, GitHub Copilot, or similar) contributed to the output — drafting code, generating test cases, writing documentation, or suggesting design options. AI-assisted does not mean unreviewed; all AI-assisted output is human-reviewed before it enters the codebase. See [AI-Assisted Workflow](ai-assisted-workflow.md).

**AI Change Log**
A record attached to a pull request or change set that documents which parts of the work were AI-assisted, which tools were used, and what human review was applied. See the [AI change log template](../templates/ai-change-log.md).

**Blast Radius**
The scope of systems, users, or data that would be affected if a change fails in production. A change with a large blast radius requires more validation, a staged rollout, or a feature flag. See [Risk Management](risk-management.md).

**Canary (Canary Release)**
A deployment strategy that sends a small slice of real traffic to a new version of a service while the rest continues hitting the old version, allowing issues to surface with limited blast radius before a full rollout.

**Circuit Breaker**
A resilience pattern that monitors failure rates on a downstream dependency and, when a threshold is breached, stops sending requests to that dependency for a cool-down period rather than letting failures cascade.

**Contract Test**
A test that verifies an API consumer and provider agree on the shape and behavior of their interface. Used to catch integration failures before they reach end-to-end or production environments. See [Testing Strategy](testing-strategy.md).

**Dead-Letter Queue (DLQ)**
A secondary queue that receives messages or jobs that have exhausted their retry attempts, preserving them for inspection, manual replay, or alerting rather than silently dropping them.

**Definition of Done**
The explicit set of conditions a unit of work must satisfy before it is considered complete. Typically includes passing tests, a completed checklist, peer review approval, and deployment to at least a staging environment. See [Release Readiness](release-readiness.md).

**Delivery Gate**
A defined checkpoint in the delivery lifecycle where a human explicitly approves or blocks progress. Every stage in the [core loop](methodology-overview.md) has at least one delivery gate. Gates are not automatic — they require a human decision.

**Evidence**
Concrete, verifiable output that a unit of work meets its acceptance criteria. Evidence includes test run results, deployment logs, screenshots of verified behavior, and completed checklists. Assertions without evidence are not acceptable as proof of completeness. See [Validation Framework](validation-framework.md).

**Fail-Open**
A safety posture where a system continues to allow requests through when a protective control (such as a rate limiter or auth check) is unavailable or errors, trading security or enforcement for availability — the opposite of fail-closed.

**Feature Flag**
A runtime configuration switch that enables or disables a feature in production without a code deployment. Used to decouple release from deployment and to limit blast radius during rollouts. See [Release Readiness](release-readiness.md).

**Hallucinated API**
An API endpoint, method signature, parameter name, or behavior that an AI model generates with confidence but that does not exist in the actual library, service, or specification being used. Hallucinated APIs are one of the most common failure modes of AI-generated code and must be caught during validation. See [Validation Framework](validation-framework.md).

**Human-in-the-Loop**
The practice of requiring a human decision at specific points in an automated or AI-assisted process. In this workflow, humans are in the loop at every delivery gate — the loop is not considered complete without their input. See [Human-in-the-Loop](human-in-the-loop.md).

**Idempotency / Idempotency Key**
The property of an operation where executing it more than once produces the same result as executing it once; an idempotency key is a unique token included with a request so that retries can be detected and deduplicated safely.

**Incident**
An unplanned disruption or degradation of a production system that affects users or violates a service level objective. Incidents trigger a postmortem. See [Incident Postmortem template](../templates/incident-postmortem.md).

**Observability**
The ability to understand the internal state of a system from its external outputs — logs, metrics, traces, and alerts. A system is observable when on-call can diagnose a failure from instrumentation alone, without code changes or guesswork. See [Security Practices](security-practices.md) and [Release Readiness](release-readiness.md).

**On-Call**
The engineer assigned responsibility for production health during a given window. On-call owns the first response to alerts and incidents, and confirms deployment health during releases.

**Postmortem**
A structured analysis of an incident conducted after it is resolved. The goal is to understand what happened, why, and what changes would prevent recurrence or reduce impact. Postmortems are blameless by default. See the [Incident Postmortem template](../templates/incident-postmortem.md) and [sample postmortem](../examples/sample-postmortem.md).

**Production Readiness**
The state of a system or change where monitoring, alerting, rollback capability, documentation, and operational runbooks are in place before traffic is directed at it. See [Production Readiness checklist](../checklists/production-readiness.md).

**Release Readiness**
The state of a change where all validation evidence, approvals, rollback plans, and release criteria are satisfied and documented. Release readiness is checked before any deployment to production. See [Release Readiness](release-readiness.md) and the [release readiness checklist](../checklists/release-readiness.md).

**Rollback**
The act of reverting a deployed change to a previous known-good state. A rollback plan is defined before every production deployment, not after an incident begins. See [Principles](principles.md) and [Release Plan template](../templates/release-plan.md).

**Runbook**
A documented, step-by-step procedure for a repeatable operational task — deploying a service, rotating credentials, responding to a specific alert. Runbooks exist so that any qualified engineer can execute the procedure without needing to locate the original author. See the [Runbook template](../templates/runbook.md).

**SLO (Service Level Objective)**
A target for a specific reliability or performance metric — such as 99.9% availability or p99 latency under 200 ms — against which a service is measured; breaching an SLO triggers an incident review. See [Incident](../templates/incident-postmortem.md).

**Staged Rollout**
A deployment strategy that gradually increases the percentage of traffic or users directed to a new version, allowing monitoring between increments. Used to limit blast radius and catch regressions before full rollout. See [Release Readiness](release-readiness.md).

**Validation**
The process of checking that a change satisfies its acceptance criteria — that it does what it is supposed to do. Validation is distinct from verification. See [Validation Framework](validation-framework.md).

**Verification**
The process of checking that a change is technically correct — that it compiles, passes type checks, passes automated tests, and meets code quality standards. Verification is necessary but not sufficient; validation is also required. See [Validation Framework](validation-framework.md).
