# Distridbug

**Detect. Investigate. Resolve. Learn.**

Distridbug is a developer-focused application observability and incident
management platform. It helps engineering teams monitor services,
identify unhealthy behaviour, manage incidents, and investigate failures
with contextual assistance.

The project is being developed as a practical, end-to-end engineering
product. Its focus is not only on application features, but also on
maintainable architecture, reliable background processing, measurable
performance, secure data handling, and production-oriented operations.

> **Project status:** Early development --- product requirements and
> architecture are being established. Features described below are
> planned unless explicitly marked as implemented.

------------------------------------------------------------------------

## Table of contents

-   [Problem](#problem)
-   [Product vision](#product-vision)
-   [Goals](#goals)
-   [Target users](#target-users)
-   [Planned capabilities](#planned-capabilities)
-   [MVP scope](#mvp-scope)
-   [High-level architecture](#high-level-architecture)
-   [Technology direction](#technology-direction)
-   [Engineering principles](#engineering-principles)
-   [Development roadmap](#development-roadmap)
-   [Local development](#local-development)
-   [Documentation](#documentation)
-   [Project status](#project-status)
-   [Contributing](#contributing)
-   [License](#license)

## Problem

When an application or one of its dependencies starts failing, engineers
often need to investigate several disconnected sources of information:

-   Health-check results and uptime history
-   Request latency and error rates
-   Application logs and service behaviour
-   Alerts and incident timelines
-   Deployment or configuration changes
-   Operational runbooks and troubleshooting notes

This investigation can be slow and reactive. Small teams may not have
the time or resources to operate a large observability stack, while
still needing a dependable way to understand what is happening in their
applications.

## Product vision

Distridbug aims to provide a lightweight, developer-friendly place to
monitor application health and coordinate incident investigation.

The platform is intended to:

1.  Monitor registered application services.
2.  Collect and retain useful health and performance signals.
3.  Evaluate configurable alert conditions.
4.  Turn relevant alerts into trackable incidents.
5.  Present incident timelines and supporting telemetry.
6.  Help engineers investigate incidents using contextual AI assistance.
7.  Measure reliability and make operational behaviour visible.

AI-generated analysis is intended to assist engineers, not replace
operational judgement. Detection and incident-management workflows
should continue to function if an external AI provider is unavailable.

## Goals

### Product goals

-   Make it straightforward to register and monitor an application.
-   Detect service failures and configurable threshold breaches.
-   Give engineers a clear view of active and historical incidents.
-   Provide enough context to support investigation and recovery.
-   Keep the initial product approachable for individual developers and
    small teams.

### Engineering goals

-   Build a modular, testable application with clear boundaries.
-   Use asynchronous processing for recurring monitoring and alert work.
-   Make background jobs recoverable and safe to retry.
-   Apply authentication and organization-level authorization.
-   Instrument the platform so its own health and performance can be
    measured.
-   Validate performance and reliability with repeatable tests.
-   Document important architectural decisions and operational
    procedures.

## Target users

The initial audience is:

-   Individual developers running web applications or APIs
-   Small engineering teams and startups
-   Backend and platform engineers
-   DevOps and site reliability practitioners

The first release is intentionally scoped for small-scale use.
Enterprise-scale capabilities and multi-region operation are not initial
requirements.

## Planned capabilities

  -----------------------------------------------------------------------
  Area                                Planned capability
  ----------------------------------- -----------------------------------
  Service registry                    Register, update, view, and remove
                                      monitored services

  Health monitoring                   Run scheduled HTTP checks with
                                      timeouts

  Telemetry                           Record check results, response
                                      times, and status codes

  Alerting                            Evaluate configurable thresholds
                                      and generate alerts

  Incident management                 Create, acknowledge, assign, and
                                      resolve incidents

  Notifications                       Notify users about relevant alerts
                                      and incidents

  Dashboard                           View service health, recent checks,
                                      and incident history

  Logs                                Collect and inspect relevant
                                      structured application logs

  AI assistance                       Summarize incident context and
                                      suggest investigation steps

  Reliability analytics               Calculate selected availability and
                                      incident metrics

  Access control                      Authenticate users and isolate
                                      organization data
  -----------------------------------------------------------------------

## MVP scope

The first release will focus on a complete, demonstrable workflow rather
than a large feature set.

### Included in the MVP

-   User registration and authentication
-   Initial organization and service management
-   Periodic HTTP health checks
-   Persistent health-check history and response-time measurements
-   Configurable threshold-based alert rules
-   Asynchronous monitoring and alert processing
-   Incident creation, acknowledgement, and resolution
-   A dashboard for services, alerts, and incidents
-   Structured application logging
-   AI-assisted incident summaries based on available incident context
-   Automated tests, containerized local development, and an initial
    cloud deployment

### Deferred beyond the MVP

The following are possible future extensions, not commitments for the
first release:

-   Kubernetes-based deployment and orchestration
-   Kafka or other higher-throughput streaming infrastructure
-   Multi-region and high-availability deployment
-   Advanced forecasting and predictive analytics
-   Autonomous production remediation
-   Enterprise single sign-on and advanced billing
-   Large-scale distributed tracing storage

These capabilities will be introduced only when product needs and
measured constraints justify them.

## High-level architecture

The initial architecture will use a **modular monolith with
independently running background workers**.

This approach keeps early development and deployment manageable while
preserving clear module boundaries. Components can be extracted into
separate services later if scale, ownership, or operational requirements
make that worthwhile.

``` mermaid
flowchart TB
    U[Developer / Team] --> FE[Next.js Web Application]
    FE --> API[FastAPI Application]

    API --> AUTH[Authentication and Access Control]
    API --> SERVICES[Service Registry]
    API --> ALERTS[Alert and Incident APIs]
    API --> ANALYTICS[Analytics APIs]

    API --> DB[(PostgreSQL)]
    API --> CACHE[(Redis)]

    SCHED[Monitoring Scheduler] --> MQ[Message Broker]
    MQ --> MW[Monitoring Workers]
    MW --> TARGET[Registered Applications]
    MW --> DB

    DB --> AW[Alert Processing Worker]
    AW --> DB
    AW --> NOTIFY[Notification Provider]

    DB --> AI[AI Analysis Worker]
    AI --> LLM[External LLM Provider]
    AI --> DB

    API --> OTEL[OpenTelemetry Instrumentation]
    MW --> OTEL
    AW --> OTEL
    OTEL --> OBS[Prometheus / Grafana]
```

### Main components

-   **Web application:** Provides service, alert, incident, and
    analytics interfaces.
-   **API application:** Handles authentication, authorization, business
    operations, and client requests.
-   **PostgreSQL:** Stores users, organizations, service configuration,
    health-check history, alerts, and incidents.
-   **Redis:** Supports caching and selected coordination or
    background-processing needs.
-   **Message broker and workers:** Execute scheduled checks and
    asynchronous processing outside request-response paths.
-   **AI analysis worker:** Builds incident context and requests
    optional AI-generated summaries and investigation suggestions.
-   **Observability stack:** Measures Distridbug's own application and
    infrastructure behaviour.

The diagram is a conceptual view. Exact communication patterns,
scheduling mechanisms, and deployment topology will be documented as
implementation decisions are made.

## Technology direction

The initial technology choices are:

  -----------------------------------------------------------------------
  Layer                   Technology              Intended use
  ----------------------- ----------------------- -----------------------
  Frontend                Next.js, React,         Web interface
                          TypeScript              

  Styling                 Tailwind CSS            UI styling

  Backend                 Python, FastAPI         APIs and business logic

  Database                PostgreSQL              Persistent relational
                                                  data

  ORM and migrations      SQLAlchemy, Alembic     Data access and schema
                                                  evolution

  Cache                   Redis                   Caching and
                                                  coordination

  Messaging               RabbitMQ                Asynchronous job
                                                  delivery

  Background processing   Celery                  Monitoring and
                                                  processing workers

  AI                      OpenAI API              Incident summaries and
                                                  investigation
                                                  assistance

  ML                      Scikit-learn            Initial
                                                  anomaly-detection
                                                  experiments

  Testing                 Pytest, Playwright      Backend and end-to-end
                                                  tests

  Containers              Docker, Docker Compose  Reproducible
                                                  environments

  CI/CD                   GitHub Actions          Automated checks and
                                                  deployment workflows

  Cloud                   AWS                     Initial hosted
                                                  deployment

  Infrastructure as Code  Terraform               Repeatable
                                                  infrastructure
                                                  provisioning

  Metrics and dashboards  Prometheus, Grafana     Platform monitoring

  Telemetry               OpenTelemetry           Metrics, logs, and
  instrumentation                                 traces instrumentation

  Load testing            Locust                  Performance and
                                                  capacity experiments
  -----------------------------------------------------------------------

The stack may evolve as implementation reveals actual requirements. A
technology will be adopted because it solves a product or engineering
problem, not simply to increase the number of tools in the repository.

## Engineering principles

Distridbug is being built around the following principles:

1.  **Start with the problem.** Every feature and dependency should have
    a clear purpose.
2.  **Prefer simple architecture first.** Introduce distributed
    complexity only when there is a reason.
3.  **Keep modules cohesive.** Separate business capabilities and make
    dependencies explicit.
4.  **Design for failure.** Account for timeouts, retries, duplicate
    messages, unavailable dependencies, and worker restarts.
5.  **Make operations observable.** Instrument the application and use
    evidence to understand its behaviour.
6.  **Secure by default.** Apply authentication, authorization, secret
    management, and organization-level data isolation.
7.  **Test meaningful behaviour.** Combine unit, integration,
    end-to-end, and failure-oriented tests.
8.  **Measure before optimizing.** Record workloads, latency,
    throughput, and resource usage before making performance claims.
9.  **Use AI responsibly.** Ground generated analysis in available
    evidence and keep humans in control of operational actions.
10. **Document decisions.** Maintain architecture notes, API
    documentation, and operational runbooks.

## Development roadmap

The initial target is an eight-week learning and implementation cycle.
The schedule is a plan and may be adjusted based on implementation
findings.

  -----------------------------------------------------------------------
  Phase                   Focus                   Expected outcome
  ----------------------- ----------------------- -----------------------
  Week 1                  Product foundation and  Requirements,
                          system design           architecture, database
                                                  design, repository
                                                  setup

  Week 2                  Backend and database    Authentication, service
                                                  registry, API
                                                  contracts, migrations,
                                                  tests

  Week 3                  Monitoring engine       Scheduler, workers,
                                                  health checks,
                                                  persisted results

  Week 4                  Alerts, incidents, and  Alert rules, incident
                          UI                      lifecycle, usable
                                                  dashboard

  Week 5                  AI and ML experiments   Contextual incident
                                                  analysis and evaluated
                                                  anomaly-detection
                                                  prototype

  Week 6                  DevOps and deployment   Containers, CI/CD,
                                                  initial AWS deployment

  Week 7                  Reliability and         Instrumentation, load
                          performance             tests, failure tests,
                                                  SLI/SLO definitions

  Week 8                  Product maturity        Security review,
                                                  documentation, UX
                                                  polish, reproducible
                                                  demo
  -----------------------------------------------------------------------

## Local development

Local setup instructions will be added as the application and its
development environment are implemented.

The intended local workflow will use:

-   Git for version control
-   Docker Compose for local infrastructure
-   A Python virtual environment for backend development
-   Node.js tooling for frontend development
-   Environment variables for local configuration

Do not commit secrets, API keys, credentials, or production
configuration to the repository. A safe example environment file will be
provided as the project setup is implemented.

## Documentation

Project documentation will live in the `docs/` directory.

  -----------------------------------------------------------------------
  Document                            Purpose
  ----------------------------------- -----------------------------------
  `docs/PRD.md`                       Product requirements, users, use
                                      cases, and acceptance criteria

  `docs/ARCHITECTURE.md`              System architecture, component
                                      boundaries, and design decisions

  `docs/DATABASE.md`                  Entities, relationships,
                                      constraints, and data-retention
                                      considerations

  `docs/adr/`                         Architecture Decision Records

  `docs/runbooks/`                    Operational and troubleshooting
                                      procedures

  `docs/api/`                         API contracts and integration notes
  -----------------------------------------------------------------------

Documentation will evolve alongside the implementation. Planned features
will be clearly distinguished from implemented behaviour.

## Project status

**Current stage:** Product definition and architecture planning.

The initial implementation will establish the repository structure,
local development environment, database foundation, and first API
endpoints.

Progress, completed features, known limitations, and future work will be
reflected here as the project develops.

## Contributing

This project is currently being developed as an engineering learning and
product-building initiative.

Contributions, bug reports, and suggestions are welcome. Before
contributing, please open an issue to discuss substantial changes.
Development guidelines and contribution instructions will be added as
the codebase matures.

## License

A license has not yet been selected. Until a license is added to the
repository, all rights are reserved by the copyright holder; reuse and
redistribution are not granted by this README.
