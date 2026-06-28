# Dispatch Service

## Responsibilities

The dispatch service is responsible for the task lifecycle: receiving task creation requests, finding the optimal available robot, assigning the task, and tracking completion or failure.

- **Task queue management** — maintains a priority queue of pending tasks per tenant
- **Robot assignment** — queries the telemetry service (via gRPC) for robot positions and availability, then assigns the closest suitable robot
- **Route optimization** — simple nearest-available assignment; advanced routing (A* pathfinding across warehouse grid) is scoped for Q4 2026
- **Task retry logic** — failed tasks are retried up to 3 times with exponential backoff (10s, 30s, 90s) before being marked FAILED
- **OTA update coordination** — coordinates with fleet-registry to schedule firmware update tasks, ensuring no more than 10% of a fleet is updating simultaneously

## Technology

- **Language:** Go 1.22
- **Database:** PostgreSQL (`dispatch_db`) — `tasks` table, `assignments` table, `task_events` audit table
- **Cache:** Redis — robot availability cache (`robot:{robot_id}:available`), populated from `acme.telemetry.robot.heartbeat` events

## Events Published

| Topic | When |
|---|---|
| `acme.dispatch.task.assigned` | A task has been successfully assigned to a robot |
| `acme.dispatch.task.completed` | A robot has reported task completion (via heartbeat `current_task_id` clearing) |
| `acme.dispatch.task.failed` | A task exhausted all retries without completion |

## Events Consumed

| Topic | What Dispatch Does With It |
|---|---|
| `acme.fleet.robot.registered` | Adds robot to the available pool in Redis |
| `acme.telemetry.robot.heartbeat` | Updates robot position and battery in Redis availability cache |
| `acme.telemetry.robot.alert` | Removes robot from available pool on CRITICAL or ERROR alerts |
| `acme.auth.user.deactivated` | Cancels all PENDING tasks for the deactivated user's tenant |

## gRPC Interface

The dispatch service exposes an internal gRPC API used by **no other services** except for the telemetry service's inverse — dispatch calls telemetry, not the other way around.

When assigning a task, dispatch makes a synchronous gRPC call to telemetry to query real-time robot positions and battery levels. This is the **only** synchronous inter-service call in ARCP (all others are via Kafka). The reason is latency: task assignment must complete within 500ms; an async event-driven flow would add unacceptable round-trip time. See [docs/architecture.md](../../docs/architecture.md) for the rationale.

**gRPC service definition:** `proto/dispatch/v1/assignment.proto`

## Task Priority Levels

Tasks are processed in this priority order:

| Priority | Value | Use Case |
|---|---|---|
| `CRITICAL` | 1 | Emergency tasks (robot retrieval, safety stop) |
| `HIGH` | 2 | Time-sensitive picks (SLA commitments) |
| `NORMAL` | 3 | Standard pick/place/transport |
| `LOW` | 4 | Non-urgent (recharging, maintenance positioning) |

Within the same priority level, tasks are processed FIFO.

## Known Technical Debt

The fleet-registry service still contains legacy scheduling logic (recurring task schedules) that was originally part of this service's domain. Coordinated deploys between dispatch and fleet-registry are required when modifying scheduling behavior. This will be resolved in Q3 2026 when the scheduling module is extracted from fleet-registry into dispatch. See [ADR-002](../../docs/decisions/002-monolith-to-services.md).

## Configuration

| Variable | Default | Description |
|---|---|---|
| `TELEMETRY_GRPC_ADDR` | `telemetry.arcp.svc.cluster.local:50051` | Telemetry gRPC endpoint |
| `TASK_MAX_RETRIES` | `3` | Max retry attempts before FAILED |
| `ASSIGNMENT_TIMEOUT_MS` | `500` | Max ms for robot assignment computation |
| `MIN_BATTERY_THRESHOLD` | `20.0` | Min battery % to assign a task to a robot |
