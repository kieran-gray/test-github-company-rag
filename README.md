# Acme Robotics Control Platform (ARCP)

ARCP is the cloud-based fleet management SaaS that powers Acme Robotics' industrial robot deployments. It provides real-time visibility, task orchestration, and operational control for fleets of warehouse autonomous mobile robots (AMRs), robotic arms, and conveyor integration units deployed at customer sites worldwide.

The platform is multi-tenant by design — each customer (tenant) manages their own fleet independently, with strict data isolation enforced at every layer.

## Key Features

- **Real-time telemetry ingestion** — heartbeats from every connected robot every 5 seconds, feeding live dashboards and anomaly detection
- **Task dispatching** — intelligent assignment of pick, place, transport, and maintenance tasks to available robots based on proximity, battery level, and priority
- **Obstacle avoidance coordination** — platform-level traffic management signals sent to robot firmware to prevent deadlocks in shared aisles
- **ERP/WMS integration** — webhook and event-based connectors for SAP, Oracle WMS, and Manhattan Associates; task queues populated directly from warehouse management systems
- **OAuth2 fleet authentication** — human operators authenticate via VaultGuard OAuth2; robots authenticate via mutual TLS (mTLS) device certificates
- **Multi-tenant architecture** — complete data isolation per tenant; `tenant_id` enforced at ORM layer across all services

## Technology Stack

| Layer | Technology |
|---|---|
| Backend services | Go 1.22 |
| Frontend dashboard | React 18, TypeScript |
| Primary database | PostgreSQL 15 (one schema per service) |
| Time-series (telemetry) | TimescaleDB 2.13 |
| Event streaming | Apache Kafka 3.5 |
| Caching / real-time state | Redis 7 |
| Container orchestration | Kubernetes (GKE) |
| API gateway | Kong 3.4 |
| CI/CD | GitHub Actions |
| Infrastructure | Terraform, Helm |

## Documentation

| Document | Description |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Service map, async principles, technology choices |
| [docs/authentication.md](docs/authentication.md) | Identity provider, token lifetimes, robot device auth |
| [docs/events.md](docs/events.md) | Kafka topic catalogue, naming conventions, schema registry |
| [docs/telemetry.md](docs/telemetry.md) | Heartbeat ingestion pipeline, alert thresholds, data retention |
| [docs/deployment.md](docs/deployment.md) | CI/CD pipeline, production checklist, rollback procedures |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Common failure modes and diagnostic steps |
| [docs/decisions/](docs/decisions/) | Architecture Decision Records (ADRs) |

## Services

| Service | Path | Purpose |
|---|---|---|
| auth | [services/auth/](services/auth/) | Human login, device cert issuance, JWKS cache |
| fleet-registry | [services/fleet-registry/](services/fleet-registry/) | Robot inventory, state machine, tenant ownership |
| dispatch | [services/dispatch/](services/dispatch/) | Task assignment, routing, retry logic |
| telemetry | [services/telemetry/](services/telemetry/) | Heartbeat ingestion, anomaly detection, alerts |
| billing | [services/billing/](services/billing/) | Invoicing, Stripe payments, usage metering |
| notifications | [services/notifications/](services/notifications/) | Email, SMS, and WebSocket push notifications |

## Local Development

Prerequisites: Docker 24+, Docker Compose v2.

```bash
# Start all services and dependencies
docker compose up

# Run with test fixtures pre-loaded
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# Run integration tests
docker compose -f docker-compose.test.yml up --abort-on-container-exit
```

The local stack starts:
- All six microservices
- PostgreSQL 15 (one database per service, initialized via `db/migrations/`)
- TimescaleDB (for telemetry service)
- Kafka + Zookeeper + Confluent Schema Registry
- Redis 7
- Kong API Gateway (available at `http://localhost:8000`)
- React dashboard (available at `http://localhost:3000`)

Default admin credentials for local dev: `admin@acme-local.dev` / `changeme123` (do not use in staging or production).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) before opening a pull request. All platform team members must follow branch naming, commit message, and review requirements described there.

