# Changelog

All notable changes to the Acme Robotics Control Platform are documented here.
Versions follow [Semantic Versioning](https://semver.org/).

---

## v3.1.0 — 2026-05-15

### Added
- **Fleet-wide OTA firmware update support.** Operators can schedule a rolling OTA firmware push to all robots in a fleet via the dispatch service. Dispatch sequences updates to ensure no more than 10% of the fleet is offline for updates simultaneously, preventing operational disruption.
- **Kafka consumer retry with exponential backoff.** Dispatch and billing service Kafka consumers now retry failed event processing up to 5 times with jitter (1s, 2s, 4s, 8s, 16s with ±20% jitter) before publishing to the dead letter queue. Previously, a processing failure caused the consumer pod to crash and restart, creating unnecessary consumer group rebalances.
- **Robot offline alert.** Telemetry service now publishes `acme.telemetry.robot.alert` with type `ROBOT_OFFLINE` if a robot fails to send a heartbeat for more than 30 seconds. Dispatch removes these robots from the available pool automatically.

### Changed
- **Migrated identity provider from Auth0 to VaultGuard** (see [ADR-003](docs/decisions/003-auth-provider.md)). Human operator authentication and robot device certificate issuance are now fully handled by VaultGuard. The Auth0 tenant is frozen.
- Access token lifetime reduced from 60 minutes to **15 minutes** (VaultGuard recommended security posture).
- Refresh token lifetime reduced from 90 days to **7 days**.

### Fixed
- Billing service was double-counting robot-months when a robot transitioned through `MAINTENANCE` state more than once in a billing cycle. Root cause: the `robot_active_days` deduplication used `(robot_id, date)` but not state — fixed to use a distinct count of robot-days regardless of state transitions.

---

## v3.0.0 — 2026-01-10

### Breaking Changes
- **Monolith decomposed into microservices.** The `arcp-monolith` binary is no longer built or deployed. All functionality is now in the six independent services under `services/`. Existing API clients must update base URLs to route through Kong API Gateway at `https://app.arcp.acme.io`.
- Database schema ownership changed. Each service now owns its own PostgreSQL database. The shared monolith schema has been split. See migration notes in `db/migrations/v3.0.0/`.
- Kafka replaces RabbitMQ as the message broker for all inter-service communication.

### Added
- Multi-tenancy enforced at ORM layer. Every database table has a `tenant_id` column with PostgreSQL row-level security policy.
- Dispatch, fleet-registry, and telemetry introduced as standalone services.
- TimescaleDB for telemetry time-series storage (see [ADR-004](docs/decisions/004-timescaledb-for-telemetry.md)).
- Kong API Gateway with JWT validation and per-tenant rate limiting.
- Confluent Schema Registry for Kafka event schema enforcement.

---

## v2.9.2 — 2025-11-03

### Fixed
- **Hotfix:** RabbitMQ consumer in the legacy telemetry module had a goroutine leak causing memory growth under sustained load. AMQP channels were not closed on consumer shutdown. Fixed with explicit channel cleanup in the shutdown handler.

  > **Note:** RabbitMQ is being replaced by Kafka in v3.0.0 (see [ADR-001](docs/decisions/001-use-kafka.md)). This fix is a stop-gap for the v2.x line only. RabbitMQ will be decommissioned in January 2026.

---

## v2.8.0 — 2025-08-20

### Added
- **Billing service** introduced as a standalone microservice, extracted from the monolith billing module. Stripe integration for credit card and ACH payments.
- Usage metering: billing counts active robot-days per tenant per billing cycle.

### Changed
- Notifications service updated to consume `acme.billing.invoice.created` and send invoice emails via SendGrid.

---

## v2.5.0 — 2025-03-01

### Deprecated
- **Auth0 integration deprecated.** Auth0 will be replaced by VaultGuard (see [ADR-003](docs/decisions/003-auth-provider.md)). Auth0 remains functional in v2.5.x but will not receive further updates. Migration planned for November 2025.

### Added
- Admin audit log: state-changing actions by `PLATFORM_ADMIN` and `TENANT_ADMIN` roles are written to an immutable audit trail in PostgreSQL.
- Per-tenant rate limiting via Kong plugin (default: 1,000 requests/minute per tenant).

---

## v2.2.0 — 2025-01-15

### Added
- Notifications service introduced (email via SendGrid, SMS via Twilio, in-app WebSocket push).
- Kafka introduced alongside RabbitMQ for new notification events.

### Changed
- PostgreSQL index optimizations on `tasks` and `robot_assignments` tables. p99 task assignment query latency reduced by ~40%.

---

## v2.0.0 — 2024-06-01

### Initial SaaS Release
- Monolithic Go application with embedded telemetry, dispatch, auth, and billing modules.
- RabbitMQ for internal messaging.
- Auth0 for identity management.
- PostgreSQL shared database (single schema).
- Manual deployment via SSH to GCE instances (see [docs/archived/old-deployment.md](docs/archived/old-deployment.md)).
