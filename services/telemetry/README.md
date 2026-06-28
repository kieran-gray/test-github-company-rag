# Telemetry Service

## Responsibilities

The telemetry service is the primary data ingestion point for the robot fleet. It handles:

- **Heartbeat ingestion** — receives gRPC streaming heartbeats from up to 50,000 concurrent robot connections
- **Real-time state maintenance** — writes the latest robot state to Redis for dashboard and dispatch consumption
- **Time-series persistence** — writes all heartbeats to TimescaleDB for historical analytics and billing
- **Anomaly detection** — evaluates alert rules on each heartbeat and publishes alerts to Kafka when thresholds are breached
- **gRPC query API** — serves synchronous robot state queries from the dispatch service

## Technology

- **Language:** Go 1.22
- **Primary storage:** TimescaleDB 2.13 (`telemetry_db`) — `heartbeats` hypertable, partitioned by `timestamp` in 1-hour chunks; `robot_metrics_1min` and `robot_metrics_1hr` continuous aggregates
- **Real-time cache:** Redis — `robot:{robot_id}:state` (latest heartbeat fields, TTL 10s)
- **Protocol:** gRPC (for both robot ingestion and dispatch queries)

See [docs/telemetry.md](../../docs/telemetry.md) for detailed pipeline documentation.

## gRPC Endpoints

| Service | Method | Caller | Description |
|---|---|---|---|
| `TelemetryService` | `ReportHeartbeat` | Robot firmware | Persistent streaming RPC; robots stream heartbeats |
| `TelemetryService` | `GetRobotState` | Dispatch service | Unary RPC; returns current robot state from Redis |
| `TelemetryService` | `GetRobotsNear` | Dispatch service | Unary RPC; returns robots within radius of a coordinate with battery above threshold |

The `GetRobotsNear` method is what dispatch uses to find the optimal robot for a task assignment. It reads exclusively from Redis (real-time state), not TimescaleDB.

**gRPC service definitions:** `proto/telemetry/v1/telemetry.proto`

## Events Published

| Topic | When |
|---|---|
| `acme.telemetry.robot.heartbeat` | Every 5 seconds per connected robot |
| `acme.telemetry.robot.alert` | Alert condition detected (low battery, error code, offline) |

## Events Consumed

None. The telemetry service receives data directly from robots via gRPC. It is a pure producer from the Kafka perspective.

## Alert Rules

| Condition | Alert Type | Severity |
|---|---|---|
| `battery_level < 15.0` | `LOW_BATTERY` | WARNING |
| `battery_level < 5.0` | `CRITICAL_BATTERY` | CRITICAL |
| `len(error_codes) > 0` | `ERROR_CODE` | ERROR |
| No heartbeat for > 30 seconds | `ROBOT_OFFLINE` | CRITICAL |

## Scaling

The telemetry service is horizontally scaled. Each pod handles a partition of the robot fleet using a consistent hashing approach: robots are assigned to a telemetry pod based on `hash(robot_id) % pod_count`. This ensures a robot's heartbeats always go to the same pod, which simplifies the "no heartbeat for 30 seconds" detection (each pod tracks its own robots).

Pod scaling is managed by the Kubernetes HPA based on active gRPC connection count (target: 5,000 connections per pod, max pods: 20).

## Configuration

| Variable | Default | Description |
|---|---|---|
| `TIMESCALEDB_DSN` | — | TimescaleDB connection string |
| `REDIS_ADDR` | `redis.internal:6379` | Redis address |
| `GRPC_PORT` | `50051` | gRPC listen port |
| `ALERT_BATTERY_WARNING` | `15.0` | Battery % threshold for LOW_BATTERY alert |
| `ALERT_BATTERY_CRITICAL` | `5.0` | Battery % threshold for CRITICAL_BATTERY alert |
| `OFFLINE_TIMEOUT_SECONDS` | `30` | Seconds without heartbeat before ROBOT_OFFLINE alert |
| `HEARTBEAT_RETENTION_DAYS` | `30` | Raw heartbeat retention in TimescaleDB |

## See Also

- [docs/telemetry.md](../../docs/telemetry.md) — full telemetry pipeline documentation
- [decisions/004-timescaledb-for-telemetry.md](../../docs/decisions/004-timescaledb-for-telemetry.md) — why TimescaleDB was chosen
