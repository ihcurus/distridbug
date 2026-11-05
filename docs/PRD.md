# Distridbug --- Product Requirements Document (PRD)

**Document status:** Draft for MVP development\
**Product:** Distridbug\
**Version:** 0.1\
**Last updated:** 2026-11-04\
**Target delivery:** Initial 8-week development cycle

------------------------------------------------------------------------

## 1. Purpose

This document defines the product requirements for Distridbug, a
developer-focused application observability and incident management
platform. It is the working contract for product and engineering
decisions during the initial development cycle. Requirements may evolve
as implementation and user feedback reveal new information.

## 2. Product summary

Distridbug helps developers and small engineering teams monitor
application services, detect unhealthy behaviour, manage incidents, and
investigate failures with relevant operational context.

The initial product will allow a user to: 1. Create an account and an
organization. 2. Register an application service and its health-check
endpoint. 3. Run scheduled HTTP health checks. 4. View current service
health and historical check results. 5. Configure alert rules. 6.
Receive alerts when conditions are breached. 7. Track incidents from
creation through resolution. 8. Request an AI-generated incident summary
and investigation suggestions based on available telemetry and logs.

AI assistance is advisory. Distridbug must not autonomously modify or
restart customer production systems in the MVP.

## 3. Problem statement

When an application or dependency becomes unhealthy, engineers often
inspect disconnected sources of information: health checks, latency
measurements, logs, alerts, deployment history, and runbooks. For small
teams, this investigation can be slow and reactive. Distridbug aims to
reduce this fragmentation by bringing basic service monitoring,
alerting, incident tracking, and contextual investigation into one
lightweight platform.

## 4. Product goals and success measures

### 4.1 Product goals

-   Make it easy to register and monitor an HTTP-based service.
-   Detect service unavailability and configured threshold breaches.
-   Present useful service health and incident history.
-   Provide a clear incident lifecycle and timeline.
-   Help engineers investigate incidents using relevant evidence.
-   Remain approachable for individual developers and small teams.

### 4.2 Engineering goals

-   Maintain clear module boundaries in a modular monolith.
-   Move recurring and potentially slow work to background workers.
-   Handle transient failures, retries, duplicate messages, and worker
    restarts.
-   Enforce organization-level authorization and data isolation.
-   Instrument the platform's own application and infrastructure.
-   Validate behaviour through automated tests and repeatable failure
    scenarios.
-   Measure performance before making scalability claims.

### 4.3 Initial success measures

These are proposed targets and must be validated under a documented test
environment.

  -----------------------------------------------------------------------
  Measure                             Initial target or definition
  ----------------------------------- -----------------------------------
  Service registration                A valid service can be registered
                                      and monitored

  Failure detection                   Detect an unavailable endpoint
                                      within two configured monitoring
                                      intervals, excluding scheduler
                                      delay and timeout

  Check persistence                   Every completed check records
                                      timestamp, outcome, and duration

  Alert generation                    A matching rule produces an alert
                                      within its configured evaluation
                                      window

  Incident deduplication              Repeated evaluations for the same
                                      active condition do not create
                                      duplicate active incidents

  Recovery                            A healthy check after failure can
                                      transition the condition toward
                                      recovery

  Dashboard API latency               p95 below 300 ms for ordinary read
                                      endpoints under a documented
                                      baseline workload

  Worker recovery                     Unacknowledged or retryable work
                                      can be processed after a worker
                                      restart

  Tenant isolation                    Users cannot access another
                                      organization's records without
                                      authorization

  AI grounding                        Analysis references supplied
                                      incident context and does not claim
                                      unavailable evidence
  -----------------------------------------------------------------------

## 5. Target users and personas

### Persona A --- Individual developer

**Needs:** Monitor a personal project or API, know when an endpoint
becomes unavailable, inspect uptime and latency.\
**Pain points:** Does not want to operate a complex observability stack
and may not have a dedicated SRE team.

### Persona B --- Small engineering team member

**Needs:** Monitor several services, share visibility, track incident
ownership, understand recent failures.\
**Pain points:** Operational information is scattered and incidents may
be handled informally.

### Persona C --- DevOps / SRE practitioner

**Needs:** Understand service health, configure actionable alerts,
measure detection and resolution, inspect evidence.\
**Pain points:** Noisy alerts and incomplete context slow down
diagnosis.

## 6. Scope

### 6.1 In scope for the MVP

1.  User registration and authentication.
2.  Initial organization creation and membership.
3.  Service registration and configuration.
4.  Scheduled HTTP health checks.
5.  Persistent health-check history.
6.  Basic latency and availability views.
7.  Threshold-based alert rules.
8.  Asynchronous alert processing and notification delivery.
9.  Incident creation, acknowledgement, assignment, and resolution.
10. Incident event timeline.
11. Structured application logs for relevant workflows.
12. AI-assisted incident summaries and investigation suggestions.
13. Basic authorization and organization-level data isolation.
14. Automated tests and containerized local development.
15. Initial cloud deployment and basic observability.

### 6.2 Explicitly out of scope for the MVP

-   Autonomous remediation or execution of production commands.
-   Kubernetes-based production deployment.
-   Multi-region high availability.
-   Kafka or a high-throughput streaming platform.
-   Enterprise SSO and complex billing.
-   Advanced predictive forecasting.
-   Full distributed tracing storage and query capabilities.
-   Native mobile applications.
-   Complex on-call scheduling and escalation policies.

These may be considered after the MVP has been validated.

## 7. Functional requirements

Priority definitions: **P0 --- Must have**, **P1 --- Should have**, **P2
--- Could have**.

### FR-01: User authentication --- P0

The system shall allow users to register, authenticate, and end their
sessions securely.

Requirements: - Register with an email address and password. - Log in
with valid credentials. - Invalid credentials must not reveal whether an
account exists. - Protected API operations require valid
authentication. - Passwords must never be stored in plaintext.

Acceptance criteria: - Valid registration succeeds. - Duplicate
registration is handled safely. - Invalid login returns an appropriate
generic error. - Unauthenticated requests to protected endpoints are
rejected. - Password hashes, not raw passwords, are stored.

### FR-02: Organization and membership --- P0

The system shall associate users and monitored resources with an
organization.

Requirements: - A new user can create an initial organization. - The
creator receives an owner role. - Membership records associate users
with organizations. - Organization-owned resources are scoped to the
active organization. - Initial roles include at least owner and member.

Acceptance criteria: - An authenticated user can access their
organization. - A user cannot read or modify another organization's
resources by changing an identifier. - Role checks are enforced by the
backend, not only the frontend.

### FR-03: Service registry --- P0

The system shall allow authorized users to register and manage HTTP
services.

A service should include display name, health-check URL, optional
description, monitoring interval, request timeout, expected successful
status, enabled/disabled state, and owning organization.

Requirements: - Create, list, view, update, enable, disable, and delete
services subject to authorization. - Reject malformed URLs and invalid
monitoring settings. - Do not expose secrets in API responses. -
Validate destinations to mitigate server-side request forgery (SSRF).
Private, loopback, link-local, and cloud metadata destinations should be
blocked by default, with explicit safe configuration for local
development and controlled tests.

Acceptance criteria: - A valid service can be registered. - Invalid
schemes and malformed URLs are rejected. - Users cannot manage another
organization's service. - Disabled services are not scheduled for new
checks. - Historical record retention after deletion follows a
documented policy.

### FR-04: Scheduled health checks --- P0

The system shall periodically execute HTTP health checks for enabled
services.

Requirements: - A scheduler creates monitoring work according to each
service's interval. - Workers execute checks outside the API
request-response lifecycle. - Each check records service, timestamp,
outcome, HTTP status where available, response duration, and a safe
error category. - Each request uses a configured timeout. - Transient
failures may be retried using a bounded policy. - A failed target must
not block checks for unrelated services. - Overlapping or excessive
scheduling for one service is controlled by a documented policy.

Acceptance criteria: - Enabled services receive checks approximately at
their configured interval, subject to scheduler and queue delay. - An
unavailable endpoint produces a failed check record. - A slow endpoint
times out within the configured timeout plus reasonable processing
overhead. - One failing target does not stop checks for others. - Worker
restart does not silently lose acknowledged work.

### FR-05: Health and telemetry views --- P0

The system shall expose recent service health and historical monitoring
results.

Requirements: - Display the latest observed service state. - Display
recent health-check history. - Display response-time measurements over a
selected time range. - Distinguish unknown, healthy, degraded, and
unavailable states where evidence supports them. - Display the latest
check timestamp. - Indicate stale data when no recent check has
completed.

Acceptance criteria: - Users can view latest results for authorized
services. - Historical checks are ordered by timestamp. - A service with
no completed checks is not incorrectly shown as healthy. - Stale or
missing telemetry is distinguishable from confirmed healthy status.

### FR-06: Alert rules --- P0

The system shall allow authorized users to configure threshold-based
alert rules.

Initial rule types: - Consecutive failed health checks. - Response
duration above a configured threshold. - Error or failure ratio above a
configured threshold, if sufficient request data exists.

Requirements: - Rules belong to an organization and may target a service
or supported service group. - Rules can be enabled or disabled. - Rule
parameters are validated. - Evaluation occurs asynchronously or as part
of the monitoring event workflow. - The rule engine records why a rule
matched. - Repeated matching evaluations are deduplicated within an
active condition.

Acceptance criteria: - A valid rule can be created, updated, enabled,
and disabled. - A matching condition creates an alert. - A non-matching
condition does not create an alert. - Repeated processing of the same
event does not create duplicate alerts. - The alert records the rule and
supporting evidence.

### FR-07: Alert lifecycle and notifications --- P0/P1

Alert records are P0; an external notification channel is P1.

Requirements: - Alerts have a defined lifecycle, such as open,
acknowledged, resolved, or suppressed. - Records include severity,
source rule, affected service, timestamps, and relevant evidence. -
Users can acknowledge alerts. - Recovery can resolve an alert according
to its recovery policy. - Support at least one notification channel for
the MVP, such as email or a development webhook. - Notification failures
are recorded and retried safely where appropriate.

Acceptance criteria: - Users can view and acknowledge alerts. - Recovery
can resolve the corresponding alert. - Notification failure does not
discard the alert. - Repeated notification attempts do not create
duplicate alert records.

### FR-08: Incident management --- P0

The system shall provide a trackable workflow for operational incidents.

Requirements: - An incident can be created from an alert. - Related
alerts may be associated with an existing active incident according to a
documented grouping policy. - An incident includes title, description,
severity, affected service, status, timestamps, and optional assignee. -
Users can acknowledge, assign, and resolve incidents according to
role. - State changes create immutable timeline events. - Resolution
notes may be recorded. - Incident and alert states remain distinct, with
explicit synchronization rules.

Suggested incident states: Open, Acknowledged, Investigating, Resolved.

Acceptance criteria: - A qualifying alert creates or updates an
incident. - Repeated events for the same active condition do not create
duplicate incidents. - Authorized users can update status and
assignment. - Every state transition is recorded in the timeline. -
Resolved incidents retain their history.

### FR-09: Dashboard and user interface --- P0

The web application shall provide a usable interface for core workflows.

Initial screens: - Sign in and registration - Organization overview -
Service list and service detail - Alert list and alert detail - Incident
list and incident detail - Basic reliability overview

Requirements: - Communicate loading, empty, success, and error states. -
Confirm destructive actions. - Allow navigation from a service to its
alerts and incidents. - Do not rely on colour alone to communicate
status. - Enforce authorization on the backend for all displayed data.

Acceptance criteria: - Users can complete the core workflow without
manually calling APIs. - Empty organizations display useful
onboarding. - Errors are understandable and do not expose stack
traces. - The UI does not display another organization's data.

### FR-10: Structured logging --- P1

Requirements: - Logs include timestamp, severity, component, and
correlation or trace identifier where available. - Credentials, tokens,
and sensitive values are not logged. - Errors include useful
troubleshooting context without exposing secrets. - Logs are available
in local development and initial deployment.

Acceptance criteria: - Important monitoring and incident workflows emit
structured logs. - Requests and jobs can be correlated across components
where context is propagated. - Secret values are excluded.

### FR-11: AI-assisted incident analysis --- P1

The system shall offer optional AI-generated incident summaries and
investigation suggestions.

Input context may include incident metadata and timeline, related
alerts, recent health checks, selected logs, and retrieved runbooks.

Requirements: - Analysis is triggered by an authorized user or defined
workflow. - Output is grounded in supplied context. - Output
distinguishes observed facts from hypotheses and recommendations. -
References to supporting evidence are included where practical. -
Provider errors, timeouts, and rate limits are handled. - AI
unavailability does not interrupt monitoring, alerting, or incident
management. - AI output cannot execute commands or modify monitored
systems.

Acceptance criteria: - An authorized user can request an incident
analysis. - The result is stored with creation time and provider/model
metadata where available. - Provider failure produces a recoverable
error state. - AI-generated suggestions are clearly labelled and not
presented as verified root cause. - The system does not claim to have
inspected evidence that was not supplied.

### FR-12: Basic anomaly-detection experiment --- P1

The project may include an initial ML prototype using historical or
generated telemetry.

Requirements: - Begin with a documented statistical baseline, such as
rolling statistics. - Evaluate a model such as Isolation Forest only
when enough suitable data exists. - Keep training and evaluation data
separate. - Record model version and evaluation results. - Compare the
model against a baseline. - Do not create high-severity incidents solely
from an unvalidated prediction.

Acceptance criteria: - The experiment has a reproducible dataset and
evaluation procedure. - Precision, recall, false-positive behaviour, and
limitations are documented where labels permit. - Model predictions are
distinguishable from deterministic alert conditions.

### FR-13: Reliability analytics --- P1

Initial measures: - Availability over a selected time window. - Number
of incidents. - Mean time to detect (MTTD), where timestamps are
available. - Mean time to resolve (MTTR), where timestamps are
available. - Response-time percentiles when there are sufficient
observations.

Requirements: - Metric definitions and calculation windows are
documented. - Missing data is not silently counted as healthy time. -
Small sample sizes and incomplete windows are identified. - Users can
view metrics only for authorized organizations and services.

Acceptance criteria: - Every displayed metric has a documented
definition. - Calculations use the selected time window. - Missing or
insufficient data is presented transparently.

### FR-14: API documentation and versioning --- P0

Requirements: - Core APIs are documented through OpenAPI. - Request and
response schemas are explicit. - Validation errors use a consistent
format. - API routes use a versioning convention, initially `/api/v1`. -
Health and readiness endpoints are available for deployment checks.

Acceptance criteria: - Core endpoints appear in generated API
documentation. - Invalid payloads return structured validation errors. -
Readiness checks reflect required dependencies according to documented
policy.

## 8. Non-functional requirements

### NFR-01: Performance

-   Ordinary dashboard read endpoints target p95 latency below 300 ms
    under a documented baseline workload.
-   Monitoring checks use bounded timeouts.
-   Slow monitoring targets do not block API request handling.
-   Database queries use appropriate indexes and bounded result sizes.
-   Performance claims include test environment and workload.

### NFR-02: Reliability and fault tolerance

-   Background jobs are retryable where safe.
-   Retries are bounded and use backoff for transient failures.
-   Repeatedly failing jobs are isolated for inspection, for example
    through a dead-letter mechanism.
-   Consumers tolerate duplicate delivery through idempotent processing.
-   Worker shutdown is graceful where practical.
-   Failure of an optional AI provider does not interrupt core
    workflows.

### NFR-03: Security and privacy

-   Passwords are securely hashed.
-   Secrets are supplied through environment configuration or a secret
    manager, never committed.
-   Authorization is enforced on the backend.
-   Organization-owned data is isolated.
-   Inputs are validated and output safely encoded.
-   Monitored URL validation mitigates SSRF.
-   Sensitive information is excluded from logs and AI context unless
    explicitly required and protected.
-   External AI requests follow a documented data-handling policy.
-   Security-sensitive actions are auditable.

### NFR-04: Maintainability

-   Business logic is not tightly coupled to HTTP route handlers.
-   Modules have clear responsibilities.
-   Database schema changes use migrations.
-   Configuration is environment-based and validated at startup.
-   Important decisions and public interfaces are documented.
-   Code follows consistent formatting, linting, and typing conventions.

### NFR-05: Testability

The project should include: - Unit tests for business rules and
utilities. - Integration tests for database and message-processing
workflows. - API tests for authentication, authorization, and
validation. - End-to-end tests for core user journeys. - Failure tests
for worker restarts, unavailable dependencies, and retry behaviour. -
Load tests for selected API and monitoring workflows.

### NFR-06: Scalability

-   Monitoring work is processed independently from API requests.
-   Worker concurrency is configurable.
-   Worker count can increase without changing business behaviour.
-   Database queries and retention are reviewed as telemetry volume
    grows.
-   Scaling decisions are based on measured bottlenecks.

### NFR-07: Observability

-   Backend and worker components expose useful health information.
-   Application metrics cover request latency, errors, job outcomes, and
    queue behaviour.
-   Logs are structured and correlated where possible.
-   OpenTelemetry is used for instrumentation where appropriate.
-   Prometheus and Grafana monitor Distridbug itself.

### NFR-08: Usability and accessibility

-   Core workflows are understandable without specialist SRE knowledge.
-   The interface provides clear empty, loading, and error states.
-   Status is conveyed through text or icons as well as colour.
-   Forms validate input and explain errors.
-   Core screens are usable at common desktop and laptop sizes.

### NFR-09: Cost awareness

-   Local development is possible with open-source infrastructure and
    documented setup.
-   Cloud resources are intentionally sized and monitored.
-   AI calls are optional, bounded, and observable for usage and cost.
-   Initial hosting assumptions and estimated operating costs are
    documented when deployment is configured.

## 9. Core user journeys

### Journey A: Register and monitor a service

1.  User signs in.
2.  User selects or creates an organization.
3.  User opens the service registry.
4.  User adds a service name, URL, interval, and timeout.
5.  Backend validates and saves the configuration.
6.  Scheduler creates monitoring work.
7.  Worker executes a health check.
8.  Result is persisted.
9.  Dashboard displays latest state and history.

**Successful outcome:** The user can see whether the service responds
and inspect recent checks.

### Journey B: Detect and manage an incident

1.  A monitored service begins failing.
2.  A worker records failed checks.
3.  Alert evaluation determines whether a configured rule is breached.
4.  The alert processor creates or updates an alert.
5.  Incident grouping creates or updates an active incident.
6.  The system sends a notification through the configured channel.
7.  User opens and acknowledges the incident.
8.  User investigates and records notes.
9.  The service recovers.
10. Recovery processing updates the alert and incident according to the
    defined lifecycle.

**Successful outcome:** The failure is visible, tracked, and recoverable
without losing history.

### Journey C: Request AI incident assistance

1.  User opens an incident.
2.  User requests AI analysis.
3.  Backend verifies organization access.
4.  Analysis worker retrieves permitted incident context.
5.  Worker sends bounded context to the configured AI provider.
6.  Provider response is validated and stored.
7.  UI displays a summary, observed evidence, hypotheses, and suggested
    investigation steps.

**Successful outcome:** The engineer receives clearly labelled
assistance while retaining control of operational actions.

## 10. Initial API capability map

Exact endpoint contracts will be maintained separately as implementation
begins.

  Resource          Expected operations
  ----------------- -----------------------------------------------------
  Authentication    Register, login, logout, current user
  Organizations     Create, list, view, update
  Memberships       List members, update supported roles
  Services          Create, list, view, update, disable, delete
  Health checks     List results, view latest result
  Alert rules       Create, list, update, enable, disable
  Alerts            List, view, acknowledge, resolve where permitted
  Incidents         Create where permitted, list, view, update, resolve
  Incident events   List timeline events
  AI analysis       Request analysis, retrieve status and result
  Analytics         Retrieve availability and incident summaries
  Platform health   Liveness and readiness

All organization-scoped operations must enforce authorization and tenant
isolation.

## 11. Data and domain rules

1.  Every customer-owned resource belongs to an organization.
2.  A health-check result is historical evidence and is not overwritten
    by a later check.
3.  Current service health is derived from recent monitoring evidence
    and a documented staleness policy.
4.  An alert represents a rule condition; an incident represents a
    tracked operational problem. They are related but distinct.
5.  Repeated events for an active condition are deduplicated.
6.  Incident state changes create timeline events.
7.  AI analyses retain enough metadata to understand when and how they
    were generated.
8.  Retention periods for health checks, logs, and analyses are
    documented before production use.
9.  Deletion behaviour accounts for related records and applicable
    retention requirements.

## 12. Risks and mitigations

  -----------------------------------------------------------------------
  Risk                    Potential impact        Mitigation
  ----------------------- ----------------------- -----------------------
  Scope expansion         MVP misses eight-week   Keep P0 scope fixed;
                          target                  defer advanced features

  Scheduler overload      Delayed or missed       Bound concurrency,
                          checks                  monitor queue delay,
                                                  apply scheduling limits

  Duplicate message       Duplicate alerts or     Idempotency keys and
  delivery                incidents               transactional state
                                                  changes

  Worker or broker outage Monitoring backlog      Durable queues, bounded
                                                  retries, recovery tests

  SSRF through monitored  Access to internal      Strict destination
  URLs                    infrastructure          validation; block
                                                  sensitive network
                                                  ranges

  Alert noise             Users ignore            Consecutive-failure
                          notifications           rules, grouping,
                                                  cooldowns, tuning

  Inaccurate AI output    Misleading              Ground output in
                          investigation           evidence, label
                                                  hypotheses, retain
                                                  human control

  Insufficient ML data    Unreliable anomaly      Start with baselines
                          detection               and synthetic
                                                  evaluation; document
                                                  limits

  Cloud cost growth       Unexpected expense      Resource limits, usage
                                                  monitoring, budget
                                                  alerts

  Tenant isolation defect Data exposure           Backend authorization
                                                  tests and
                                                  organization-scoped
                                                  queries
  -----------------------------------------------------------------------

## 13. Assumptions and constraints

### Assumptions

-   Initial monitored targets expose HTTP or HTTPS health-check
    endpoints.
-   First users are individual developers and small teams.
-   First deployment is single-region.
-   A third-party AI provider may be used for optional analysis.
-   Controlled demo applications will support repeatable testing.
-   Initial development cycle is approximately eight weeks.

### Constraints

-   Development time is limited.
-   Infrastructure and AI usage should remain cost-conscious.
-   The MVP should be maintainable by a small engineering team.
-   Kubernetes is not required to run the initial product.
-   The product must not execute autonomous production remediation.

## 14. MVP release acceptance checklist

The MVP is ready for demonstration when these outcomes are verified:

-   [ ] A user can register, sign in, and create or access an
    organization.
-   [ ] Organization-level authorization is enforced by the backend.
-   [ ] A user can register an HTTP service.
-   [ ] The scheduler creates monitoring work for enabled services.
-   [ ] Workers execute checks with bounded timeouts.
-   [ ] Check outcomes and response durations are persisted.
-   [ ] The dashboard displays current and historical service health.
-   [ ] A configured alert rule can be triggered by a controlled
    failure.
-   [ ] Repeated processing does not create duplicate active alerts or
    incidents.
-   [ ] A user can acknowledge and resolve an incident.
-   [ ] Incident state changes appear in its timeline.
-   [ ] Recovery is detected and reflected in relevant records.
-   [ ] Optional AI analysis summarizes supplied incident context.
-   [ ] Core workflows continue when the AI provider is unavailable.
-   [ ] Automated tests cover critical business and authorization rules.
-   [ ] The application runs locally using documented instructions.
-   [ ] A repeatable deployment process is available.
-   [ ] Basic application and worker telemetry is visible.
-   [ ] A controlled failure-and-recovery demonstration is documented.

## 15. Open product and technical decisions

  -----------------------------------------------------------------------
  Decision                            Initial direction
  ----------------------------------- -----------------------------------
  Authentication mechanism            Choose and document a secure
                                      session or token-based approach

  Organization onboarding             Create an initial organization
                                      during signup

  Monitoring scheduler                Select a scheduler compatible with
                                      the worker architecture

  Queue semantics                     Define acknowledgement, retry, and
                                      dead-letter behaviour

  Incident grouping                   Start with service- and
                                      alert-condition-based grouping

  Notification channel                Select one simple MVP channel

  Health-state staleness              Define based on monitoring interval
                                      and timeout

  Telemetry retention                 Set initial limits appropriate for
                                      the MVP

  AI provider and model               Select based on quality, latency,
                                      cost, and data handling

  Anomaly detection                   Evaluate a statistical baseline
                                      before adopting a model
  -----------------------------------------------------------------------

Material decisions should be recorded in Architecture Decision Records
(ADRs).

## 16. Change management

This PRD is a living document. When a requirement changes:

1.  Describe the problem or new information.
2.  Identify affected requirements and modules.
3.  Assess impact on scope, schedule, architecture, security, and
    testing.
4.  Record the decision and rationale.
5.  Update acceptance criteria and related documentation.

A feature is not complete solely because its code exists. It must
satisfy its acceptance criteria and have appropriate tests and
documentation.

------------------------------------------------------------------------

**End of document**
