# Architecture

This document describes the high-level architecture of the Acme Robotics Control Platform (ARCP). For historical context on key decisions, see the [ADRs in docs/decisions/](decisions/).

## Service Map

ARCP is composed of six microservices, each with its own PostgreSQL database (or TimescaleDB). Services communicate primarily via Kafka events. The only exception is the gRPC interface between the telemetry service and the dispatch service, retained for latency-sensitive robot state queries (see below).

```
                        ┌─────────────────────────────────────────────┐
                        │              Kong API Gateway                │
                        │           (kong.internal:8000)              │
                        └──────┬──────┬──────┬──────┬────────────────┘
                               │      │      │      │
              ┌────────────────┘      │      │      └────────────────┐
              │                       │      │                        │
        ┌─────▼──────┐        ┌───────▼──┐  │  ┌──────────┐  ┌──────▼─────┐
        │    auth    │        │ fleet-   │  │  │ dispatch │  │  billing   │
        │  service   │        │ registry │  │  │ service  │  │  service   │
        └─────┬──────┘        └───────┬──┘  │  └──────┬───┘  └──────┬─────┘
              │                       │      │         │              │
              │               ┌───────▼──────▼─────────▼──────────────▼──────┐
              │               │               Apache Kafka 3.5                │
              │               │         (kafka.internal:9092)                 │
              └───────────────►                                               │
                              │  Topics: acme.fleet.*, acme.dispatch.*,      │
                              │  acme.telemetry.*, acme.billing.*,           │
                              │  acme.auth.*                                  │
                              └───────┬───────────────────┬───────────────────┘
                                      │                   │
                              ┌───────▼───┐       ┌───────▼───────┐
                              │ telemetry │       │ notifications │
                              │  service  │       │    service    │
                              └───────────┘       └───────────────┘
```

### Services

| Service | Responsibility | Database |
|---|---|---|
| auth | Human login via VaultGuard, robot device cert issuance, JWKS cache | PostgreSQL (`auth_db`) |
| fleet-registry | Robot inventory, state machine, tenant ownership | PostgreSQL (`fleet_db`) |
| dispatch | Task queue, robot assignment, routing, retry logic | PostgreSQL (`dispatch_db`) |
| telemetry | Heartbeat ingestion, anomaly detection, real-time state | TimescaleDB (`telemetry_db`) + Redis |
| billing | Invoicing, Stripe payments, usage metering | PostgreSQL (`billing_db`) |
| notifications | Email, SMS, WebSocket push | Redis only (no persistent DB) |

## Core Principles

### Each service owns its own database

No service reads directly from another service's database. Cross-service data access happens exclusively through Kafka events or, in the single approved exception, the telemetry→dispatch gRPC interface. This boundary is enforced in code review — PRs that introduce cross-schema SQL joins are rejected.

### Async-first communication

Services publish events to Kafka and subscribe to topics they care about. This decouples services and enables the replay capability that was the primary driver for choosing Kafka over RabbitMQ (see [ADR-001](decisions/001-use-kafka.md)).

### The gRPC exception

The dispatch service makes synchronous gRPC calls to the telemetry service when computing optimal robot assignments. The query is "which available robots are closest to task location X with battery > 20%?" This cannot be done asynchronously without unacceptable assignment latency. This is the **only** approved synchronous call between services. All other inter-service communication must be via Kafka.

See [docs/events.md](events.md) for the full topic catalogue.

### Multi-tenancy

Every database table has a `tenant_id` UUID column. The ORM (a thin wrapper around `database/sql` in Go) automatically appends `WHERE tenant_id = $1` to every query. Row-level security policies in PostgreSQL provide a second enforcement layer. Bypassing tenant isolation is a critical security defect.

### API Gateway

All external traffic enters through Kong 3.6, which handles:
- JWT validation (verifying against the auth service's JWKS endpoint)
- Rate limiting (per-tenant, default 1,000 req/min)
- Request routing to upstream services
- TLS termination

Internal service-to-service HTTP calls are not routed through Kong; they use Kubernetes service DNS directly (e.g. `fleet-registry.arcp.svc.cluster.local`).

## Technology Choices

- **PostgreSQL 15** — chosen for ACID compliance, row-level security, and familiarity across the team
- **TimescaleDB** — chosen for telemetry time-series; see [ADR-004](decisions/004-timescaledb-for-telemetry.md)
- **Kafka 3.5** — chosen for event ordering, replay, and telemetry throughput; see [ADR-001](decisions/001-use-kafka.md)
- **Redis 7** — session cache, JWKS cache, real-time robot state (`robot:{robot_id}:state`), rate limiting counters
- **Kong 3.6** — API gateway; chosen over Nginx/Envoy for its plugin ecosystem (rate limiting, JWT validation out of the box)

## What's Not Settled Yet

Fleet-registry still contains legacy scheduling logic inherited from the v2.x monolith. This is targeted for extraction into the dispatch service in Q3 2026. See [ADR-002](decisions/002-monolith-to-services.md) for migration status.
