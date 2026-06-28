# ADR-004: Use TimescaleDB for Telemetry Time-Series Storage

**Status:** Accepted
**Date:** 2026-02-01
**Decision makers:** Sarah Chen (Platform Architect), David Park (Telemetry Lead)

---

## Context

During the v3.0.0 launch (January 2026), the telemetry service was storing robot heartbeats in a standard PostgreSQL 15 table with range partitioning on the `timestamp` column (monthly partitions). Under the projected load of 50,000 robots × 12 heartbeats/minute, this amounted to approximately **600,000 rows/minute** or **36 million rows/hour**.

Load testing in January 2026 revealed that standard PostgreSQL partitioning was insufficient:
- Write throughput degraded after approximately 2 billion rows (around 55 hours of data at full load)
- Partition pruning was effective for point-in-time queries but failed for range scans across partition boundaries (e.g. "battery trend for robot X over the last 48 hours")
- Compression was not automatic — manual partition archiving was needed to keep storage costs manageable

Storage projections: at 30 days of raw heartbeats at full scale, the `heartbeats` table would require approximately 8 TB. This is manageable but growing toward the point where we'd need a more purpose-built solution within 6–12 months.

## Decision

Adopt **TimescaleDB 2.13** as a PostgreSQL extension for the telemetry service's heartbeat storage.

TimescaleDB operates as a PostgreSQL extension rather than a separate database system, which means:
- The telemetry service continues to use the standard `database/sql` Go driver
- TimescaleDB automatic chunk management replaces manual partition management
- Continuous aggregates enable pre-computed 1-minute and 1-hour rollups for the dashboard's historical charts
- Native compression (columnar) reduces storage by approximately 85% for chunks older than 7 days

The telemetry service's database is now `TimescaleDB` while all other services continue to use standard PostgreSQL 15. The distinction is purely in the telemetry DB deployment — the interface from application code is identical.

## Alternatives Considered

### InfluxDB

InfluxDB 3.0 was evaluated. Rejected because:
- A separate database system to operate, monitor, and back up — adds operational burden
- The team has no InfluxDB expertise
- InfluxQL / Flux query language differs from SQL, requiring separate tooling and expertise

### ClickHouse

ClickHouse was considered for the analytics path. Rejected because:
- Team unfamiliar with ClickHouse operational requirements (replication, schema migrations)
- Strong analytics performance but overkill for our read patterns — dashboard queries are simple time-range scans, not complex OLAP queries
- Adds a third distinct database system alongside PostgreSQL and Redis

### Continue with PostgreSQL + Aggressive Partitioning

Modeled 6-month and 12-month storage and query performance projections. Rejected because:
- At 12 months of growth, p99 write latency exceeded 50ms under simulated load — unacceptable for a 5-second heartbeat cycle
- Manual partition maintenance (monthly partition creation, archiving) is toil without a clear ceiling
- No automatic compression

## Consequences

- **Positive:** Automatic chunk management eliminates manual partition work. Continuous aggregates make historical dashboard queries fast without application-level caching. Native compression reduces 30-day storage from projected 8 TB to approximately 1.2 TB.
- **Positive:** No application code changes required — TimescaleDB is a PostgreSQL extension; all existing queries work.
- **Negative:** TimescaleDB adds a deployment dependency. The telemetry service's Helm chart now pulls a TimescaleDB-enabled PostgreSQL image rather than stock PostgreSQL. Operations team needed a brief onboarding session on TimescaleDB-specific maintenance tasks (tune compression policy, monitor chunk count).
- **Operational note:** TimescaleDB compression jobs run on a 1-hour schedule. Chunks older than 7 days are compressed. Monitor `timescaledb_information.chunks` for compression status.
