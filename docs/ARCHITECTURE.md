# Distridbug --- System Architecture

**Status:** Initial architecture baseline\
**Version:** 0.1\
**Last updated:** 2026-11-05\
**Related document:** [PRD.md](./PRD.md)

> This document describes the intended architecture. It is not a claim
> that all components have been implemented. Update it as decisions are
> validated during development.

## 1. Purpose and goals

Distridbug is a developer-focused application observability and incident
management platform. Its architecture must support scheduled service
monitoring, asynchronous processing, alerting, incident tracking,
optional AI assistance, and observability of the platform itself.

The architecture should: - Be achievable within the initial eight-week
development cycle. - Keep local development and initial deployment
manageable. - Separate business capabilities into cohesive modules. -
Keep recurring and slow work out of API request handlers. - Handle
timeouts, retries, duplicate messages, and partial failures. - Enforce
organization-level authorization and data isolation. - Allow future
scaling based on measured bottlenecks.

## 2. Architectural style

### Modular monolith with independent workers

The initial API and business capabilities will live in one FastAPI
application, organized into modules with explicit responsibilities.
Background workers will run as separate processes and reuse
domain/application code where appropriate.

This gives us clear module boundaries and independent worker execution
without prematurely taking on the operational complexity of many
microservices.

We will consider extracting a module only when independent scaling,
deployment, ownership, or reliability requirements justify it.

### Principles

1.  High cohesion and low coupling.
2.  Domain rules independent of FastAPI, Celery, SQLAlchemy, and
    provider SDKs.
3.  Asynchronous execution for scheduled checks and slow operations.
4.  PostgreSQL as the source of truth for business records.
5.  Idempotent consumers because message delivery can be repeated.
6.  Explicit transaction boundaries and durable event publication.
7.  Backend-enforced authorization and secure defaults.
8.  Observable behaviour and evidence-led optimization.

## 3. System context

External actors and systems: - **Developer/team member:** Uses the web
application. - **Monitored application:** Exposes an HTTP/HTTPS health
endpoint. - **Notification provider:** Delivers email or webhook
notifications. - **AI provider:** Optionally analyzes bounded incident
context.

``` mermaid
flowchart LR
    USER[Developer / Team] -->|HTTPS| DIST[Distridbug]
    DIST -->|Scheduled health checks| APP[Monitored Applications]
    DIST -->|Notifications| NOTIFY[Email / Webhook Provider]
    DIST -->|Bounded incident context| AI[External AI Provider]
```

The initial monitoring workflow does not require an agent installed in
the monitored application. Richer telemetry ingestion can be added
later.

## 4. High-level component architecture

``` mermaid
flowchart TB
    B[Browser] --> FE[Next.js Web App]
    FE -->|REST / optional WebSocket| API[FastAPI Application]

    subgraph MODULES["Modular backend"]
      AUTH[Authentication and Authorization]
      ORG[Organizations and Memberships]
      REG[Service Registry]
      MON[Monitoring]
      ALERT[Alerts]
      INC[Incidents]
      ANA[Analytics]
      AI[AI Analysis]
    end

    API --> AUTH
    API --> ORG
    API --> REG
    API --> MON
    API --> ALERT
    API --> INC
    API --> ANA
    API --> AI

    API --> DB[(PostgreSQL)]
    API --> REDIS[(Redis)]
    SCHED[Scheduler] --> MQ[(RabbitMQ)]
    MQ --> MW[Monitoring Workers]
    MQ --> AW[Alert / Notification Worker]
    MQ --> AIW[AI Analysis Worker]
    MW --> DB
    AW --> DB
    AIW --> DB
    MW --> MQ
    AW --> MQ

    API --> OTEL[OpenTelemetry]
    MW --> OTEL
    AW --> OTEL
    AIW --> OTEL
    OTEL --> PROM[Prometheus]
    PROM --> G[Grafana]
```

This is a logical view. Modules initially share one codebase and
PostgreSQL database; they are not separate microservices.

## 5. Component responsibilities

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Next.js frontend                    Screens, forms, charts, user
                                      interaction

  FastAPI application                 HTTP interface, validation,
                                      authentication, use-case
                                      orchestration

  Auth module                         Identity and session/token
                                      validation

  Organization module                 Membership and role rules

  Service registry                    Service configuration and
                                      validation

  Monitoring module                   Check rules and health-result
                                      creation

  Alert module                        Rule evaluation and alert lifecycle

  Incident module                     Incident state transitions and
                                      event timeline

  Analytics module                    Availability and incident
                                      calculations

  AI analysis module                  Context assembly and analysis
                                      workflow

  Scheduler                           Find due services and enqueue
                                      checks

  Monitoring worker                   Execute HTTP checks and persist
                                      results

  Alert worker                        Evaluate alert events and deliver
                                      notifications

  AI worker                           Run bounded AI analysis jobs

  PostgreSQL                          Durable product data

  RabbitMQ                            Asynchronous message delivery

  Redis                               Cache and selected ephemeral
                                      coordination

  Prometheus/Grafana                  Monitor Distridbug itself
  -----------------------------------------------------------------------

Redis must not be the sole store for incidents, health-check history, or
other important business records. RabbitMQ is not the long-term business
history store.

## 6. Backend module boundaries

Suggested structure:

``` text
backend/
├── app/
│   ├── main.py
│   ├── api/
│   │   ├── dependencies.py
│   │   ├── error_handlers.py
│   │   └── v1/
│   ├── core/
│   │   ├── config.py
│   │   ├── security.py
│   │   ├── logging.py
│   │   └── exceptions.py
│   ├── modules/
│   │   ├── auth/
│   │   ├── organizations/
│   │   ├── services/
│   │   ├── monitoring/
│   │   ├── alerts/
│   │   ├── incidents/
│   │   ├── analytics/
│   │   └── ai_analysis/
│   ├── infrastructure/
│   │   ├── database/
│   │   ├── messaging/
│   │   ├── cache/
│   │   └── providers/
│   └── workers/
│       ├── scheduler.py
│       ├── monitoring_worker.py
│       ├── alert_worker.py
│       └── ai_worker.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── contract/
└── alembic/
```

Within a module, use layers where they clarify responsibilities:

-   **Domain:** Entities, value objects, domain rules, domain
    interfaces.
-   **Application:** Use cases, orchestration, transaction boundaries.
-   **Infrastructure:** Database repositories, message publishers,
    provider adapters.
-   **API:** HTTP routes and request/response schemas.

Keep route handlers thin. Do not create abstractions for every function;
introduce them when they protect a meaningful boundary or improve
testability.

``` mermaid
flowchart TB
    API[API delivery layer] --> APP[Application use cases]
    WORKER[Worker delivery layer] --> APP
    APP --> DOMAIN[Domain rules]
    APP --> PORTS[Required interfaces]
    INFRA[Infrastructure adapters] --> PORTS
```

The domain must not import FastAPI, Celery, SQLAlchemy models, or
external provider SDKs.

## 7. Synchronous request flow

1.  Browser sends an HTTPS request.
2.  API authenticates the user.
3.  Backend authorizes access to the requested organization/resource.
4.  Request schema validates input.
5.  An application use case performs the business operation.
6.  A repository reads or writes data within a transaction.
7.  API returns a documented response.
8.  Logs and telemetry record the operation without sensitive values.

``` mermaid
sequenceDiagram
    actor User
    participant UI as Next.js
    participant API as FastAPI
    participant Auth as Access Control
    participant UC as Application Use Case
    participant DB as PostgreSQL

    User->>UI: Register monitored service
    UI->>API: POST /api/v1/services
    API->>Auth: Authenticate and authorize
    Auth-->>API: User and organization context
    API->>UC: Register service
    UC->>DB: Validate ownership and persist
    DB-->>UC: Stored service
    UC-->>API: Result
    API-->>UI: 201 Created
    UI-->>User: Show service
```

Service registration must not synchronously wait for a health check.
Monitoring is a separate workflow.

## 8. Monitoring and asynchronous processing

### Why a worker architecture?

Health checks are recurring work and monitored targets may be slow or
unavailable. A slow target must not consume API capacity or block checks
for unrelated services.

### Monitoring flow

``` mermaid
sequenceDiagram
    participant Scheduler
    participant MQ as RabbitMQ
    participant Worker as Monitoring Worker
    participant Target as Monitored Service
    participant DB as PostgreSQL
    participant Alert as Alert Processor

    Scheduler->>DB: Find due enabled services
    Scheduler->>MQ: Publish check job
    MQ-->>Worker: Deliver job
    Worker->>Target: HTTP request with timeout
    Target-->>Worker: Response or timeout
    Worker->>DB: Persist check result
    Worker->>MQ: Publish check.completed
    MQ-->>Alert: Deliver event
    Alert->>DB: Evaluate rules and update alert
```

### Scheduler responsibilities

-   Find enabled services whose next check is due.
-   Enqueue a job with a unique job/check identifier.
-   Avoid excessive duplicate work for the same service.
-   Recover due work after a scheduler restart.
-   Track scheduling delay where practical.

The scheduler does not determine that a check succeeded. A persisted
worker result is the evidence.

### Monitoring worker responsibilities

1.  Receive the job.
2.  Load current service configuration.
3.  Confirm that the service remains enabled.
4.  Validate the target destination using the SSRF policy.
5.  Execute HTTP with connection and total timeouts.
6.  Capture status, duration, and a safe error category.
7.  Persist the result.
8.  Publish a completion event after the result is committed.
9.  Acknowledge the job only after the durable result is committed.

Do not store full response bodies by default; they may contain sensitive
or unexpectedly large data.

### Delivery, idempotency, and retries

Assume messages can be delivered more than once.

-   Give jobs/events unique identifiers.
-   Use database uniqueness constraints or idempotency keys at the
    persistence boundary.
-   Acknowledge messages only after durable effects are committed.
-   Make consumers safe to run again after a crash.
-   Retry only transient failures.
-   Use bounded exponential backoff with jitter.
-   Send repeatedly failing jobs to a dead-letter queue or equivalent
    failure store.
-   Record safe failure metadata and provide an inspection/replay
    procedure.

A monitored endpoint timeout is normally a failed check result, not a
reason to hammer the target with immediate retries. Monitoring cadence
and transport retries are separate policies.

## 9. Alerts and incidents

### Alert evaluation

A completed check or supported telemetry event can trigger rule
evaluation. The alert engine: - Loads enabled rules applicable to the
service. - Evaluates the event against each rule. - Records why a rule
matched. - Creates or updates an alert using an idempotent condition
key. - Resolves or transitions the alert when its recovery condition is
met.

### Incident grouping

An incident is a tracked operational problem, not simply another name
for an alert.

Initial grouping should consider organization, affected service, alert
condition/rule family, and active incident state. The grouping policy
must be explicit so unrelated failures are not merged.

### Incident lifecycle

``` mermaid
stateDiagram-v2
    [*] --> Open
    Open --> Acknowledged
    Acknowledged --> Investigating
    Investigating --> Resolved
    Open --> Resolved
    Acknowledged --> Resolved
    Resolved --> [*]
```

Every valid transition must be authorized, checked against current
state, persisted transactionally, and recorded as an incident timeline
event with actor and timestamp.

Whether a later failure reopens an incident or creates a new one is an
open product decision to resolve before implementing recovery semantics.

## 10. Data architecture

### PostgreSQL

PostgreSQL is the source of truth for: - Users, organizations, and
memberships. - Monitored service configuration. - Health-check
results. - Alert rules and alerts. - Incidents and incident events. - AI
analysis metadata/results. - Audit records where required.

Use Alembic migrations. Define foreign keys, uniqueness constraints,
indexes, and transaction boundaries deliberately.

### Redis

Potential uses: - Rebuildable cache entries. - Short-lived
coordination. - Rate-limit counters. - Selected scheduler/worker
coordination.

Do not store the only copy of important business records in Redis.

### RabbitMQ

Initial logical queues may include:

``` text
monitoring.check
monitoring.check.completed
alerts.evaluate
notifications.send
ai.analysis
```

Exact exchanges, routing keys, durability, dead-letter configuration,
and Celery integration will be decided during implementation.

### Transactional event publishing

A database commit can succeed while message publication fails, or a
message can be published before the database transaction commits.

For important workflows, consider a transactional outbox:

1.  Persist the business change and an outbox event in one database
    transaction.
2.  A publisher reads pending outbox records.
3.  Publish the event to RabbitMQ.
4.  Mark the outbox record as published.
5.  Consumers remain idempotent because publication may repeat.

Introduce the outbox for the most important workflows first rather than
adding it indiscriminately.

## 11. AI and ML architecture

### AI incident analysis

AI analysis is optional and asynchronous.

``` mermaid
sequenceDiagram
    actor User
    participant API as FastAPI
    participant MQ as RabbitMQ
    participant Worker as AI Worker
    participant DB as PostgreSQL
    participant LLM as AI Provider

    User->>API: Request analysis
    API->>DB: Verify access and create request
    API->>MQ: Enqueue analysis job
    API-->>User: Pending
    MQ-->>Worker: Deliver job
    Worker->>DB: Retrieve permitted incident context
    Worker->>LLM: Send bounded, redacted context
    LLM-->>Worker: Structured response
    Worker->>DB: Validate and persist analysis
    User->>API: Retrieve result
    API->>DB: Read result
    API-->>User: Summary and suggestions
```

Context may include incident metadata/timeline, related alerts, recent
checks, selected redacted logs, and relevant runbooks.

Controls: - Do not send secrets or unnecessary personal data. - Bound
context size, request duration, and cost. - Validate provider output
against a schema. - Separate observed facts from hypotheses and
recommendations. - Store provider/model metadata where available. -
Handle timeouts, rate limits, and provider outages. - Never execute
model-generated commands in the MVP. - Core monitoring and incident
workflows must work without AI.

### ML anomaly detection

ML is an experiment, not a critical dependency.

1.  Establish a statistical baseline using rolling windows.
2.  Generate or collect representative telemetry.
3.  Define anomaly labels or evaluation assumptions.
4.  Evaluate a candidate such as Isolation Forest.
5.  Compare it with the baseline.
6.  Record precision, recall, false positives, and limitations where
    labels permit.
7.  Version model and evaluation artifacts.

An anomaly score is not proof of an incident. Initially, predictions are
advisory and must be distinguishable from deterministic alert
conditions.

## 12. API design

Initial route groups (proposals, not implemented endpoints):

``` text
/api/v1/auth
/api/v1/organizations
/api/v1/services
/api/v1/services/{service_id}/health-checks
/api/v1/alert-rules
/api/v1/alerts
/api/v1/incidents
/api/v1/incidents/{incident_id}/events
/api/v1/incidents/{incident_id}/analyses
/api/v1/analytics
/health/live
/health/ready
```

Conventions: - Resource-oriented HTTP methods. - Explicit
request/response schemas. - Consistent validation and error responses. -
Pagination for potentially large collections. - Backend authorization
for every organization-scoped operation. - OpenAPI documentation. - Do
not expose database models directly.

Example error envelope:

``` json
{
  "error": {
    "code": "SERVICE_NOT_FOUND",
    "message": "The requested service was not found.",
    "request_id": "request-correlation-id"
  }
}
```

Do not expose stack traces, SQL details, secrets, or raw provider errors
to clients.

## 13. Security architecture

### Authentication and authorization

-   Choose and document a secure session or token approach, including
    expiry and revocation.
-   Hash passwords with a modern password-hashing algorithm.
-   Enforce permissions in backend use cases/dependencies.
-   Verify organization membership for every organization-scoped
    resource.
-   Never trust a client-supplied organization ID without membership
    validation.
-   Test cross-tenant access attempts.

### SSRF protection

User-supplied URLs are requested by our infrastructure, creating SSRF
risk.

Controls: - Allow only HTTP and HTTPS. - Reject credentials embedded in
URLs. - Block loopback, private, link-local, multicast, reserved, and
cloud metadata destinations by default. - Resolve and validate DNS
results. - Revalidate destinations after redirects, or disable redirects
initially. - Defend against DNS rebinding by ensuring the connected
destination is the validated destination. - Apply network-level egress
restrictions in deployment. - Use a controlled allowlist for local test
targets rather than weakening production validation.

URL validation alone is insufficient; network egress controls add
another security boundary.

### Secrets and data handling

-   Keep local secrets in ignored environment files.
-   Use a cloud secret manager or protected deployment secrets for
    hosted environments.
-   Never commit credential-bearing `.env` files.
-   Redact tokens and sensitive payloads from logs.
-   Define what incident data may be sent to an AI provider.
-   Establish retention and deletion policies.

## 14. Observability of Distridbug

Distridbug must monitor itself.

### Structured logs

Include timestamp, severity, component, environment, request/trace
identifier, and job/event identifier where applicable. Never log
passwords, tokens, authorization headers, or unnecessary response
bodies.

### Metrics

Initial platform metrics: - HTTP request count, duration, and errors. -
Database latency and connection-pool usage. - Queue depth and message
age. - Worker task duration, success, failure, and retries. - Scheduler
lag and number of due checks. - Monitoring execution duration and
outcomes. - AI request count, latency, errors, and usage/cost where
available.

Avoid high-cardinality Prometheus labels such as raw user IDs, full
URLs, and incident IDs.

### Traces

Use OpenTelemetry to correlate API requests, database operations,
message publication/consumption, worker execution, and external provider
calls. Propagate trace context across asynchronous boundaries where
supported.

### Candidate SLIs

  -----------------------------------------------------------------------
  SLI                                 Definition
  ----------------------------------- -----------------------------------
  API availability                    Successful eligible API requests /
                                      eligible requests

  API latency                         Distribution of API response
                                      duration

  Monitoring freshness                Time since last completed check
                                      compared with expected schedule

  Job processing delay                Time between enqueue and worker
                                      start

  Job success rate                    Successful jobs, with retries
                                      reported separately

  Incident detection delay            Time between first qualifying
                                      evidence and alert/incident
                                      creation
  -----------------------------------------------------------------------

Set SLO targets after measuring a baseline. Distridbug's own
availability is separate from the availability of applications it
monitors.

## 15. Deployment architecture

### Local development

Docker Compose should run the application and supporting infrastructure
locally.

``` mermaid
flowchart TB
    DEV[Developer machine]
    FE[Next.js]
    API[FastAPI]
    WORKERS[Celery workers]
    SCHED[Scheduler]
    DB[(PostgreSQL)]
    REDIS[(Redis)]
    MQ[(RabbitMQ)]
    PROM[Prometheus]
    GRAFANA[Grafana]

    DEV --> FE
    DEV --> API
    DEV --> WORKERS
    DEV --> SCHED
    API --> DB
    API --> REDIS
    SCHED --> MQ
    WORKERS --> MQ
    WORKERS --> DB
    PROM --> API
    PROM --> WORKERS
    GRAFANA --> PROM
```

Tests should be runnable locally without a cloud account. External AI
calls should be configurable and optional.

### Initial cloud direction

Potential components: - AWS compute for frontend, API, scheduler, and
workers. - Managed or self-hosted PostgreSQL depending on budget and
operational needs. - Redis and RabbitMQ with persistence and restricted
network access. - S3 for selected artifacts or backups. - Managed secret
storage or protected deployment secrets. - HTTPS termination and
restrictive security groups.

Choose exact AWS services and topology in a deployment ADR after
reviewing cost and operational requirements. Not every component needs a
separate cloud service for the MVP.

### CI/CD

A GitHub Actions pipeline should progressively include: 1. Formatting
and lint checks. 2. Static type checks where configured. 3. Unit tests.
4. Integration tests with service containers. 5. Frontend build and
tests. 6. Container image build. 7. Dependency/image security checks. 8.
Deployment after required checks pass.

Use protected secrets. Treat database migrations as an explicit
deployment step.

## 16. Failure handling

  -----------------------------------------------------------------------
  Failure                             Expected behaviour
  ----------------------------------- -----------------------------------
  Monitored endpoint times out        Record failed check; continue
                                      unrelated work

  Monitoring worker crashes           Broker redelivers unacknowledged
                                      work according to configuration

  RabbitMQ unavailable                Queue-dependent operations fail
                                      visibly and remain recoverable;
                                      unrelated reads may continue

  PostgreSQL unavailable              Readiness becomes unhealthy; do not
                                      acknowledge work whose durable
                                      result was not committed

  Redis unavailable                   Redis-dependent features degrade
                                      explicitly; durable records remain
                                      in PostgreSQL

  AI provider unavailable             Analysis fails/retries safely; core
                                      monitoring continues

  Duplicate event                     Idempotency prevents duplicate
                                      business effects

  Notification provider unavailable   Preserve alert/incident and retry
                                      notification within bounds

  Scheduler restarts                  Rediscover due checks without
                                      unbounded duplicate work
  -----------------------------------------------------------------------

Do not return success from an operation if its required durable action
has not occurred.

## 17. Scalability and evolution

Initial scaling levers: - Increase monitoring worker concurrency within
safe limits. - Run additional worker replicas. - Tune queue prefetch and
task timeouts. - Add database indexes based on query plans. - Bound
dashboard query windows and paginate results. - Aggregate older
telemetry. - Apply retention and archival policies. - Cache read-heavy
data where correctness permits.

Signals for future architectural changes:

  -----------------------------------------------------------------------
  Observed constraint                 Possible response
  ----------------------------------- -----------------------------------
  Monitoring jobs delay API work      Increase or isolate worker capacity

  A module needs independent scaling  Extract behind a stable API/event
                                      contract

  Database write volume becomes a     Partition or move high-volume
  bottleneck                          telemetry to a purpose-built store

  Queue throughput is insufficient    Revisit broker topology or evaluate
                                      streaming infrastructure

  Deployment coordination becomes     Establish stronger service
  difficult                           boundaries

  Availability needs exceed one host  Add redundancy and managed
                                      infrastructure
  -----------------------------------------------------------------------

Kubernetes, Kafka, microservices, and multi-region deployment should be
adopted only when a measured requirement justifies their cost.

## 18. Architecture Decision Records

Store material decisions in `docs/adr/`.

Suggested initial ADRs:

  ADR       Decision
  --------- ------------------------------------------------------
  ADR-001   Modular monolith with independent background workers
  ADR-002   PostgreSQL as primary source of truth
  ADR-003   RabbitMQ for initial asynchronous processing
  ADR-004   Docker Compose for local development
  ADR-005   Organization-level multi-tenancy
  ADR-006   Asynchronous AI analysis with human oversight
  ADR-007   Monitoring URL validation and SSRF protections

Each ADR should include context, decision, alternatives,
consequences/trade-offs, status, and date.

## 19. Implementation sequence

1.  Establish repository structure, configuration, and local development
    environment.
2.  Add PostgreSQL, SQLAlchemy, Alembic, and initial domain modules.
3.  Implement authentication, organizations, and service registry.
4.  Add scheduler, RabbitMQ, and monitoring worker.
5.  Persist health-check results and expose history APIs.
6.  Add alert evaluation and incident lifecycle.
7.  Build frontend workflows.
8.  Add optional AI analysis and anomaly-detection experiment.
9.  Add instrumentation, load tests, and failure tests.
10. Deploy and document the operating environment.

Build incrementally. Prefer a thin end-to-end vertical slice that can be
run and tested over building all infrastructure before product behaviour
exists.

------------------------------------------------------------------------

**End of document**
