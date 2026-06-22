# Production Readiness Checklist

Use this checklist before promoting any service, feature, or significant change to production. "Production ready" means the system can be operated, observed, debugged, and recovered by someone who was not on the team that built it.

Related files: [release readiness checklist](release-readiness.md) · [pre-merge checklist](pre-merge.md) · [release readiness doc](../docs/release-readiness.md) · [runbook template](../templates/runbook.md) · [risk assessment template](../templates/risk-assessment.md)

---

## 1. Observability

- [ ] Structured logs are emitted for all significant events: requests, errors, background job completions, and state transitions
- [ ] Every log line includes a correlation/trace ID so a single request can be followed across services
- [ ] Application metrics are exported: request rate, error rate, latency (p50/p95/p99), queue depth, and any domain-specific counters relevant to this service
- [ ] Distributed traces are instrumented for the critical paths — not just at the entry point but across downstream calls
- [ ] Logs, metrics, and traces are being received and retained in the observability stack before the change goes live (not assumed — verified)

## 2. Alerting and SLOs

- [ ] Service Level Objectives are defined: availability target, latency target, and error budget
- [ ] Alerts fire when SLOs are at risk — not only when they are already breached
- [ ] Alert routing is configured: the right team or on-call rotation receives the alert, not a shared inbox that goes unmonitored
- [ ] Alert thresholds are calibrated to avoid alert fatigue — alert only on conditions that require a human to act
- [ ] Synthetic monitors or uptime checks are in place for external-facing endpoints
- [ ] Alerts have been tested by simulating a failure in a non-production environment

## 3. Dashboards

- [ ] A service dashboard exists showing the four golden signals: latency, traffic, errors, saturation
- [ ] The dashboard is linked from the runbook so on-call can find it immediately during an incident
- [ ] Dashboard panels use consistent time ranges and include meaningful titles and units — no unlabeled graphs
- [ ] Deployment markers are shown on the dashboard so a spike can be correlated to a specific release

## 4. Runbook

- [ ] A runbook exists at [templates/runbook.md](../templates/runbook.md) and has been filled out for this service/operation
- [ ] The runbook covers: what the service does, how to start/stop/restart it, common failure modes and their remediation steps, escalation path, and links to dashboards and alert definitions
- [ ] The runbook has been reviewed by someone who was not involved in writing it — tested for clarity
- [ ] On-call knows where the runbook lives

## 5. Scaling and Capacity

- [ ] Expected peak load has been estimated: requests per second, concurrent users, data volume
- [ ] The system has been load-tested or back-of-envelope validated against the peak estimate — not assumed to scale
- [ ] Resource limits (CPU, memory, database connections, file descriptors) are set and their breach behavior is understood
- [ ] Auto-scaling policies, if present, have been tested to confirm they trigger and recover correctly
- [ ] Downstream dependencies (databases, queues, external APIs) have been checked for their own limits — the bottleneck may not be the service itself

## 6. Graceful Degradation

- [ ] The service has defined behavior when a dependency is unavailable: timeout values are set, retries have backoff and jitter, circuit breakers trip before cascading failures spread
- [ ] Non-critical features degrade gracefully when their dependencies are down — the core path continues to function
- [ ] The system returns meaningful errors to callers when it cannot serve a request, rather than hanging or returning incorrect data silently
- [ ] Health check endpoints correctly reflect dependency health — a service that can't reach its database should not report healthy

## 7. Data Safety and Backups

- [ ] Any new database tables or data stores have backup policies configured and verified (not just enabled — a test restore has been performed)
- [ ] Migrations are backwards-compatible with the current running version: the new schema works with the old code until the old code is fully drained
- [ ] Destructive operations (drops, bulk deletes, truncates) require explicit confirmation and are not triggered by automated processes without a human gate
- [ ] Sensitive data retention and deletion policies are enforced and documented

## 8. Rate Limiting and Abuse Protection

- [ ] External-facing endpoints have rate limits configured per client/IP/token — limits are documented and communicated to API consumers
- [ ] Rate limit responses return `429` with a `Retry-After` header — not a generic 5xx
- [ ] Unauthenticated endpoints have stricter limits than authenticated ones
- [ ] Expensive operations (bulk export, report generation, ML inference) have per-request cost awareness and limits

## 9. Configuration Management

- [ ] All environment-specific values (URLs, credentials, feature flags, timeouts, limits) are externalized into configuration — not hardcoded in source
- [ ] Configuration is validated at startup: the service fails fast with a clear error if required config is missing or malformed, rather than silently using a wrong default
- [ ] Configuration changes can be made without a code deployment where operationally appropriate
- [ ] Secrets are stored in a secrets manager — not in environment variable files committed to the repository

## 10. Rollback and Forward Recovery

- [ ] A rollback plan exists: the previous version can be deployed in under 10 minutes if the new version causes a production incident
- [ ] Database migrations are reversible, or a compensating migration is prepared and tested
- [ ] Feature flags are in place for significant behavioral changes so the new behavior can be disabled without a full rollback
- [ ] The deployment process has been rehearsed in a staging environment — production is not the first time this deploy sequence runs

## 11. Load and Performance Baseline

- [ ] Baseline performance metrics have been recorded before the change (latency, throughput, error rate) so post-deploy behavior can be compared
- [ ] Any known performance regressions introduced by this change are documented and accepted, with a follow-up ticket created
- [ ] Memory leak checks: long-running processes have been observed over time in staging for growing memory usage

---

## Sign-off

| Role | Name / Handle | Date |
|---|---|---|
| Engineer | | |
| Tech lead / reviewer | | |
| On-call / operator | | |

> This checklist confirms a structured readiness review was performed. It does not guarantee the system will not fail — it confirms the team is positioned to detect, respond to, and recover from failure.
