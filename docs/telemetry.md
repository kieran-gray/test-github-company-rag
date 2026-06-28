# Telemetry

## Overview

The telemetry service is responsible for ingesting real-time heartbeat data from the robot fleet, detecting alert conditions, and maintaining the authoritative real-time state of every connected robot.

At scale, ARCP supports up to 50,000 concurrent robot connections. The telemetry pipeline is designed for this throughput: robots connect directly to the telemetry service via a persistent gRPC streaming connection, bypassing the Kong API Gateway (which is not optimized for long-lived streaming connections).

## Heartbeat Protocol

Robots send a heartbeat every **5 seconds** over a persistent gRPC stream established at startup. The gRPC endpoint:

```
telemetry.arcp.svc.cluster.local:50051
grpc method: telemetry.TelemetryService/ReportHeartbeat
```

Robots authenticate this connection using their mTLS device certificate (see [docs/authentication.md](authentication.md) for the device cert flow).

### Heartbeat Payload

Each heartbeat contains:

| Field | Type | Description |
|---|---|---|
| `robot_id` | UUID | Unique robot identifier |
| `tenant_id` | UUID | Owning tenant |
| `timestamp` | ISO 8601 | Robot's local clock (synchronized via NTP) |
| `position_x` | float64 | X coordinate in meters (warehouse coordinate system) |
| `position_y` | float64 | Y coordinate in meters |
| `position_z` | float64 | Z coordinate (floor level, usually 0.0) |
| `battery_level` | float32 | Battery percentage (0.0–100.0) |
| `current_task_id` | UUID | Currently executing task, or null if idle |
| `error_codes` | []int32 | List of active error codes (empty if healthy) |
| `firmware_version` | string | Robot firmware version, e.g. `fw-2.4.1` |
| `speed_ms` | float32 | Current speed in m/s |

## Data Pipeline

```
Robot → gRPC stream → Telemetry Service → TimescaleDB (persistence)
                                        ↓
                                      Redis  (real-time state)
                                        ↓
                                     Kafka  (acme.telemetry.robot.heartbeat)
                                            (acme.telemetry.robot.alert)
```

1. Telemetry service receives the heartbeat via gRPC
2. Writes to **TimescaleDB** for persistence (raw heartbeat record in `heartbeats` hypertable, partitioned by `timestamp` in 1-hour chunks)
3. Updates **Redis** with the latest robot state (key: `robot:{robot_id}:state`, TTL: 10 seconds)
4. Publishes `acme.telemetry.robot.heartbeat` to Kafka for downstream consumers (fleet-registry last-seen updates, dispatch robot position queries)
5. Evaluates alert rules (see below). If triggered, publishes `acme.telemetry.robot.alert`

The Redis real-time state cache is what the dispatch service (via gRPC) and the dashboard query when they need the current position and availability of a robot. TimescaleDB is only queried for historical/analytics workloads.

## Alert Rules

The telemetry service evaluates the following conditions on every heartbeat:

| Condition | Alert Severity | Published Event |
|---|---|---|
| `battery_level < 15.0` | WARNING | `acme.telemetry.robot.alert` (type: `LOW_BATTERY`) |
| `battery_level < 5.0` | CRITICAL | `acme.telemetry.robot.alert` (type: `CRITICAL_BATTERY`) |
| `len(error_codes) > 0` | ERROR | `acme.telemetry.robot.alert` (type: `ERROR_CODE`) |
| No heartbeat for > 30 seconds | CRITICAL | `acme.telemetry.robot.alert` (type: `ROBOT_OFFLINE`) |

When a CRITICAL or ERROR alert is published, the dispatch service consumes it and moves the robot to the `UNAVAILABLE` state, preventing new task assignments.

## Data Retention

| Data | Storage | Retention |
|---|---|---|
| Raw heartbeats | TimescaleDB | 30 days |
| Aggregated 1-minute metrics | TimescaleDB continuous aggregate | 2 years |
| Real-time state | Redis | 10-second TTL (evicted if robot disconnects) |

TimescaleDB compression is enabled for raw heartbeats older than 7 days, reducing storage by approximately 85%.

## Dashboard Queries

The React dashboard uses two data sources:

- **Real-time view** (robot map, live battery indicators): reads from Redis via the telemetry service REST API (`GET /telemetry/robots/{robot_id}/state`)
- **Historical view** (battery trends, task history, uptime charts): queries TimescaleDB via the telemetry service analytics API (`GET /telemetry/robots/{robot_id}/history`)

Direct database access from the dashboard is not permitted — all queries go through the telemetry service API.

## See Also

- [docs/events.md](events.md) — Kafka topic details for `acme.telemetry.*`
- [services/telemetry/README.md](../services/telemetry/README.md) — service implementation
- [decisions/004-timescaledb-for-telemetry.md](decisions/004-timescaledb-for-telemetry.md) — why TimescaleDB was chosen
