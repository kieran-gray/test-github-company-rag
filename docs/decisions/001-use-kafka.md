# ADR-001: Use Apache Kafka as the Event Streaming Platform

**Status:** Accepted
**Date:** 2025-01-15
**Decision makers:** Sarah Chen (Platform Architect), Marcus Webb (Engineering Lead)

---

## Context

As ARCP began the transition from a monolith to microservices (see ADR-002), the team needed to select an asynchronous messaging platform for inter-service communication. The requirements were driven primarily by the telemetry use case:

- **High throughput:** At full scale, the platform will handle 50,000 robots sending heartbeats every 5 seconds — approximately 600,000 messages per minute on the telemetry heartbeat topic alone.
- **Event ordering:** Robot state transitions (AVAILABLE → IN_TASK → AVAILABLE) must be processed in order per robot. Out-of-order processing could cause incorrect availability calculations in the dispatch service.
- **Replay capability:** The telemetry pipeline will feed a time-series database (TimescaleDB). If the TimescaleDB write path fails, or if a new downstream consumer is added later, we need the ability to replay historical events. This is non-negotiable for the billing use case: if billing service is deployed after robots have been active, it must be able to reconstruct the history of robot registrations.
- **Durability:** Events must not be lost if a consumer is temporarily unavailable.

## Decision

Apache Kafka 3.5 will be the messaging backbone for all ARCP inter-service communication.

Kafka fulfills all four requirements: it handles our projected throughput with headroom, guarantees ordering within a partition (we will partition telemetry topics by `robot_id`), supports replay via configurable retention (default 7 days, extended to 30 days for the heartbeat topic), and provides durable storage on-disk.

The team evaluated three alternatives:

### RabbitMQ

RabbitMQ was the messaging platform used in the ARCP v2.x monolith (during the experimental multi-service period in late 2024). It was rejected for this role because:
- No native replay capability. Once a message is consumed and acknowledged, it is gone. The billing replay requirement cannot be satisfied.
- Exchange topology becomes complex at scale. The proposed RabbitMQ design (see [archived/rabbitmq-design.md](../archived/rabbitmq-design.md)) required 14 separate exchanges to handle all routing scenarios.
- Under load testing at 100,000 messages/minute, RabbitMQ consumer throughput degraded due to the per-message ACK overhead.

### AWS SQS / SNS

Amazon SQS with SNS fan-out was evaluated. Rejected because:
- Vendor lock-in. Acme Robotics has a multi-cloud policy, and committing the core messaging fabric to AWS would restrict future deployment flexibility.
- SQS retention is capped at 14 days, which constrains replay scenarios.
- No native schema registry integration.

### Redis Streams

Redis Streams were considered for the real-time telemetry path specifically. Rejected as a primary backbone because:
- Not suitable as the primary persistent event store. Redis is memory-bound; storing 30 days of heartbeat history for 50,000 robots would require impractical amounts of RAM.
- Consumer group semantics are less mature than Kafka's.
- Redis Streams remain in use for the **real-time robot state cache** (separate from the messaging backbone) — see architecture.md.

## Consequences

- **Positive:** Replay capability is available for billing and analytics use cases. Ordering guarantees per partition simplify dispatch and fleet-registry logic. Confluent Schema Registry integration provides contract enforcement between producers and consumers.
- **Negative:** Kafka adds operational complexity. The team underwent a two-week Kafka training program in January 2025. Kafka requires more infrastructure (brokers, ZooKeeper/KRaft, Schema Registry) than a comparable RabbitMQ deployment.
- **Operational note:** Kafka is managed via the Confluent Cloud managed offering in production to reduce operational burden on the platform team.

## Notes on RabbitMQ

The v2.9.2 release (November 2025) included a hotfix for a RabbitMQ consumer memory leak in a legacy component that was still running during the transition period. This was a stop-gap fix during migration. All RabbitMQ consumers were decommissioned as part of the v3.0.0 release.
