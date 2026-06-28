# Fleet Registry Service

## Responsibilities

The fleet-registry service is the source of truth for Acme Robotics' robot inventory. It tracks:

- **Robot registration** — records robot serial number, model, firmware version, and tenant ownership when a robot is first commissioned
- **State machine** — maintains each robot's lifecycle state
- **Tenant-robot ownership** — enforces that robots belong to exactly one tenant; tenant isolation at the fleet level
- **Last-seen tracking** — updates robot heartbeat timestamps from telemetry events

### Robot State Machine

```
REGISTERING → AVAILABLE → IN_TASK → AVAILABLE
                  ↓                      ↓
             MAINTENANCE → AVAILABLE
                  ↓
            DECOMMISSIONED (terminal)
```

State transitions are validated by the fleet-registry. Invalid transitions (e.g. DECOMMISSIONED → AVAILABLE) are rejected with a 400 error.

## Technology

- **Language:** Go 1.22
- **Database:** PostgreSQL (`fleet_db`) — `robots`, `robot_state_history`, `tenants`, `tenant_robot_assignments` tables
- **Cache:** Redis — current robot state cached at `robot:{robot_id}:registry_state` (separate from telemetry's `robot:{robot_id}:state`)

## REST API

| Method | Path | Description |
|---|---|---|
| `POST` | `/robots` | Register a new robot (serial number, model, tenant) |
| `GET` | `/robots/{robot_id}` | Get robot details and current state |
| `PATCH` | `/robots/{robot_id}/state` | Trigger a state transition |
| `DELETE` | `/robots/{robot_id}` | Decommission a robot (moves to DECOMMISSIONED) |
| `GET` | `/tenants/{tenant_id}/robots` | List all robots for a tenant |
| `GET` | `/tenants/{tenant_id}/robots?state=AVAILABLE` | Filter by state |

## Events Published

| Topic | When |
|---|---|
| `acme.fleet.robot.registered` | Robot successfully registered and moved to AVAILABLE |
| `acme.fleet.robot.decommissioned` | Robot moved to DECOMMISSIONED state |

## Events Consumed

| Topic | What Fleet-Registry Does With It |
|---|---|
| `acme.telemetry.robot.heartbeat` | Updates `last_seen_at` and `battery_level` on the robot record |
| `acme.auth.user.deactivated` | Flags robots owned by the deactivated user's tenant as `owner_deactivated = true` (does not decommission, requires explicit admin action) |

## Known Technical Debt

The fleet-registry service still contains a **legacy scheduling module** from the ARCP v2.x monolith era. This module handles recurring task schedules — for example, "recharge robot ARX-007 at bay 12 every 4 hours." This logic properly belongs in the dispatch service but was not extracted during the v3.0.0 migration due to time constraints.

**Impact:** Changes to recurring scheduling behavior require coordinated deploys of both fleet-registry and dispatch. The scheduling module is not independently testable without the full dispatch stack.

**Resolution:** Extraction of this module into dispatch is planned for **Q3 2026**. See [ADR-002](../../docs/decisions/002-monolith-to-services.md) for details.

## Configuration

| Variable | Default | Description |
|---|---|---|
| `FLEET_DB_DSN` | — | PostgreSQL connection string for fleet_db |
| `REDIS_ADDR` | `redis.internal:6379` | Redis address |
| `STATE_CACHE_TTL` | `30s` | Redis TTL for robot state cache entries |

## See Also

- [docs/architecture.md](../../docs/architecture.md) — service isolation principles
- [services/dispatch/README.md](../dispatch/README.md) — task assignment (related coupling)
