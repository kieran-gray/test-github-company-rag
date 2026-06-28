# Events

## Overview

Apache Kafka 3.5 is the event backbone for ARCP. Services communicate asynchronously by publishing to and consuming from Kafka topics. This is the preferred communication pattern; the only exception is the telemetry→dispatch gRPC interface (see [docs/architecture.md](architecture.md)).

For the decision to use Kafka over RabbitMQ and other alternatives, see [ADR-001](decisions/001-use-kafka.md).

## Topic Naming Convention

All ARCP topics follow this format:

```
acme.{domain}.{entity}.{verb}
```

Examples:
- `acme.fleet.robot.registered`
- `acme.dispatch.task.assigned`
- `acme.telemetry.robot.alert`

Domain is the owning service's domain. Entity is the object the event describes. Verb is past tense.

## Topic Catalogue

| Topic | Producer | Consumers | Description |
|---|---|---|---|
| `acme.fleet.robot.registered` | fleet-registry | dispatch, billing, notifications | A new robot has been registered to a tenant |
| `acme.fleet.robot.decommissioned` | fleet-registry | billing, notifications | A robot has been permanently removed from service |
| `acme.dispatch.task.assigned` | dispatch | telemetry, notifications | A task has been assigned to a specific robot |
| `acme.dispatch.task.completed` | dispatch | billing, notifications | A task has been completed successfully |
| `acme.dispatch.task.failed` | dispatch | notifications | A task failed after all retries were exhausted |
| `acme.telemetry.robot.heartbeat` | telemetry | fleet-registry, dispatch | Periodic robot heartbeat (position, battery, error codes) |
| `acme.telemetry.robot.alert` | telemetry | notifications, dispatch | Alert condition detected (low battery, error code, offline) |
| `acme.billing.invoice.created` | billing | notifications | A monthly invoice has been generated for a tenant |
| `acme.billing.payment.succeeded` | billing | notifications | A Stripe payment succeeded |
| `acme.billing.payment.failed` | billing | notifications | A Stripe payment failed |
| `acme.auth.user.created` | auth | notifications, billing | A new user account has been created |
| `acme.auth.user.deactivated` | auth | fleet-registry, dispatch, billing | A user account has been deactivated |

## Event Schema

All events must include these envelope fields:

```json
{
  "event_id": "uuid-v4",
  "event_type": "acme.fleet.robot.registered",
  "schema_version": "1.0",
  "tenant_id": "uuid",
  "timestamp": "2026-05-15T14:23:00Z",
  "payload": { ... }
}
```

`event_id` is used for idempotency deduplication by consumers. Consumers must record processed `event_id` values and skip duplicates. This is non-negotiable — Kafka delivery semantics guarantee at-least-once delivery, and consumers will see duplicate events.

Schemas are registered in Confluent Schema Registry at `https://schema-registry.internal`. All schema changes must be backward-compatible (additive only) unless a major version bump is coordinated across all consumers.

## Size Limits

**Maximum event payload size: 512 KB.**

This limit was reduced from 1 MB in February 2026 following an incident where an oversized telemetry batch caused broker log segment corruption on a single partition. If you need to transfer large blobs (e.g. robot diagnostic dumps), store them in GCS and include a signed URL in the event payload.

## Dead Letter Queues

Failed events are published to a dead letter topic:

```
acme.dlq.{original-topic-name}
```

For example, failed events from `acme.dispatch.task.assigned` go to `acme.dlq.acme.dispatch.task.assigned`.

Each service has a DLQ consumer that alerts on Slack (`#platform-alerts`) when new messages arrive. Dead letters require manual investigation and replay — see [docs/troubleshooting.md](troubleshooting.md).

## Consumer Groups

Each consumer service should use a stable consumer group ID of the form `arcp-{service-name}`. Do not use random or ephemeral consumer group IDs — they prevent Kafka from tracking committed offsets and will cause events to be reprocessed from the beginning on restart.

## Replaying Events

Kafka retains events for 7 days by default (configurable per topic). To replay events from a specific offset, use the Kafka admin CLI:

```bash
# Reset consumer group offset to beginning of a topic
kafka-consumer-groups.sh --bootstrap-server kafka.internal:9092 \
  --group arcp-billing --topic acme.dispatch.task.completed \
  --reset-offsets --to-earliest --execute
```

Replay is one of the key reasons Kafka was chosen over RabbitMQ. See [ADR-001](decisions/001-use-kafka.md).
