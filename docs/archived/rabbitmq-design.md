# RabbitMQ Design Proposal

> **ARCHIVED / REJECTED — This proposal was abandoned before full implementation.**
> **RabbitMQ is not used in the current ARCP architecture. Kafka is the event backbone.**
> **See [decisions/001-use-kafka.md](../decisions/001-use-kafka.md) for the accepted decision.**

---

## Background

In December 2024, during the early planning phase for ARCP's microservices decomposition, the original platform team (pre-Sarah Chen) proposed RabbitMQ as the messaging backbone. This document captures that proposal for historical reference.

## Proposed Exchange Topology

The proposal centered on a topic exchange architecture with the following design:

```
Exchange: arcp.events (type: topic)
  ├── Binding: fleet.# → Queue: fleet-registry-queue
  ├── Binding: dispatch.# → Queue: dispatch-queue
  ├── Binding: telemetry.# → Queue: telemetry-queue
  ├── Binding: billing.# → Queue: billing-queue
  └── Binding: # → Queue: audit-log-queue (catch-all)

Exchange: arcp.dlx (type: direct) — Dead Letter Exchange
  └── Binding: dlx.rejected → Queue: dead-letter-queue
```

Message format:
```json
{
  "routing_key": "fleet.robot.registered",
  "correlation_id": "uuid",
  "body": { ... }
}
```

## Why This Was Rejected

The proposal was reviewed by Sarah Chen upon joining as Platform Architect in January 2025. Key objections:

1. **No replay.** RabbitMQ's push model means messages are gone after acknowledgement. The billing use case — being able to reconstruct robot activity history for invoice disputes — requires replay. This was a hard requirement.

2. **Scale concerns.** Load estimates for the telemetry heartbeat path (600,000 messages/minute) were modeled against RabbitMQ. Under sustained load at this rate, the per-message ACK overhead and queue depth management became a bottleneck in testing. Kafka's sequential disk writes are better suited to this volume.

3. **14 exchanges.** The proposed topology had grown to 14 separate exchanges during the design phase, with complex routing key patterns. This was identified as an operational maintenance risk.

4. **Team familiarity.** Ironically, Kafka was chosen despite lower initial team familiarity, because the long-term fit was clearly better. A training investment was made in January 2025.

## Note on v2.9.2

RabbitMQ **was partially implemented** during the transitional period between the monolith and the full microservices architecture. A small subset of the notification pipeline used RabbitMQ queues from approximately v2.2.0 (January 2025) through v3.0.0 (January 2026).

The v2.9.2 hotfix (November 2025) addressed a **goroutine leak in the RabbitMQ consumer** in this legacy component. The AMQP channel was not being closed on consumer shutdown, causing goroutines to accumulate until the notification service OOM-crashed under sustained load. This was patched as a stop-gap; the component was fully replaced with Kafka consumers in v3.0.0.

All RabbitMQ infrastructure was decommissioned in January 2026.
