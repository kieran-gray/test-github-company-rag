# ADR-002: Decompose the Monolith into Microservices

**Status:** Accepted (2025-06-01) — Migration in progress
**Date:** 2025-06-01
**Decision makers:** Sarah Chen (Platform Architect), Marcus Webb (Engineering Lead), Priya Nair (VP Engineering)

---

## Context

ARCP v2.x was a single Go binary (`arcp-monolith`) that embedded all platform functionality: telemetry ingestion, task dispatch, fleet management, authentication, billing, and notifications. This architecture served the company well during the initial product-market fit phase (2024–early 2025), but by mid-2025, three scaling constraints became untenable:

1. **Telemetry throughput bottleneck.** The telemetry ingestion module and the billing module shared the same process. Scaling horizontally to handle more robot connections meant scaling billing pods unnecessarily — a poor resource fit. The monolith could not be scaled per-capability.

2. **Team autonomy.** The billing team (3 engineers) and the fleet team (5 engineers) were blocked on each other constantly because all work happened in a single codebase with a single deployment pipeline. A bad billing deploy could take down telemetry.

3. **Database contention.** All modules shared a single PostgreSQL instance. Long-running analytics queries from the billing module caused lock contention on tables used by the dispatch module.

## Decision

Decompose the monolith into six independent services:

| Service | Scope |
|---|---|
| `auth` | Human authentication, robot device certificates |
| `fleet-registry` | Robot inventory, ownership, state machine |
| `dispatch` | Task queue, robot assignment, routing |
| `telemetry` | Heartbeat ingestion, real-time state, anomaly detection |
| `billing` | Invoicing, payments, usage metering |
| `notifications` | Email, SMS, WebSocket push |

Each service:
- Owns its own database (PostgreSQL or TimescaleDB)
- Deploys independently via its own Helm chart
- Communicates with other services only via Kafka events (with the approved gRPC exception for dispatch→telemetry; see architecture.md)

## Migration Strategy

The team adopted a **strangler fig** pattern:
1. New features are implemented as standalone services from the outset.
2. Existing monolith modules are extracted incrementally as capacity permits.
3. The monolith's API routes are deprecated and redirected to the new services via Kong gateway configuration.

The monolith was deprecated as of the v3.0.0 release (January 2026) and is no longer built or deployed.

## Current Status (as of June 2026)

The migration is largely complete. Five of the six services are fully extracted and operating independently.

**Outstanding item:** The `fleet-registry` service still contains a legacy scheduling module that was originally part of the monolith's dispatch logic. This handles recurring task scheduling (e.g. "recharge robot X every 4 hours"). This logic was not extracted during the v3.0.0 cutover due to time constraints. It is owned by the fleet team and is targeted for extraction into the dispatch service in **Q3 2026**.

Until that extraction is complete, the dispatch service and fleet-registry have a tighter coupling than intended. Changes to recurring scheduling logic require coordinated deploys of both services.

## Consequences

- **Positive:** Each service can be scaled independently. Teams can deploy their services without coordinating with other teams. Database contention eliminated.
- **Negative:** Operational complexity increased significantly. The team now operates six services, six databases, and Kafka rather than a single binary. Distributed tracing (via OpenTelemetry → Jaeger) was added to maintain debuggability.
- **Ongoing risk:** The remaining legacy scheduling module in fleet-registry is a known coupling risk. It is tracked in the Q3 2026 roadmap.
