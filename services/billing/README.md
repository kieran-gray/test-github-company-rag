# Billing Service

## Responsibilities

The billing service is the financial system of record for ARCP. It handles:

- **Invoice generation** — creates monthly invoices for each tenant on the 1st of each month, based on active robot count and task volume
- **Stripe payment processing** — charges the tenant's payment method on file via Stripe; handles card declines, retries, and webhook confirmation
- **Usage metering** — tracks which robots were active (defined as: registered and not decommissioned) for each day of the billing cycle
- **Refunds** — processes credit notes and refunds via Stripe; requires TENANT_ADMIN approval
- **Subscription status** — is the authoritative source for whether a tenant's subscription is active. Other services (fleet-registry, dispatch) should not make billing decisions independently.

## Technology

- **Language:** Go 1.22
- **Database:** PostgreSQL (`billing_db`) — stores tenants, invoices, line items, payment records, usage meter readings
- **Payment processor:** Stripe (Go SDK v76)

## Events Published

| Topic | When |
|---|---|
| `acme.billing.invoice.created` | Monthly invoice generated for a tenant |
| `acme.billing.payment.succeeded` | Stripe payment confirmed successful |
| `acme.billing.payment.failed` | Stripe payment failed (card decline, network error) |

## Events Consumed

| Topic | What Billing Does With It |
|---|---|
| `acme.auth.user.created` | Creates a billing account (with a Stripe customer ID) for the new tenant |
| `acme.fleet.robot.registered` | Starts metering for the new robot (adds a meter record for this robot/tenant pair) |
| `acme.fleet.robot.decommissioned` | Closes the meter record for the robot; no further charges |
| `acme.dispatch.task.completed` | Records a task completion for usage-based billing items (some plans charge per task) |

All event consumers use the `event_id` field for idempotency deduplication (see [docs/events.md](../../docs/events.md)).

## Billing Model

**Default plan:** Per active robot per month. A robot is considered active for a day if it has sent at least one heartbeat in that calendar day (as recorded in the `billing_db.robot_active_days` table, populated from `acme.telemetry.robot.heartbeat` events via the metering pipeline).

**Billing cycle:** Invoices are generated at 00:00 UTC on the 1st of each month by a scheduled Kubernetes CronJob (`billing-invoice-job`). Do not deploy the billing service during the change freeze window (last 3 business days of each month) — this risks interrupting the invoicing job.

**Pricing:** Stored in `billing_db.plans` table, configurable per tenant. Default rate: $149/robot/month.

## Configuration

| Variable | Default | Description |
|---|---|---|
| `STRIPE_SECRET_KEY` | — | Stripe secret key (from GCP Secret Manager) |
| `STRIPE_WEBHOOK_SECRET` | — | Stripe webhook signing secret |
| `BILLING_CURRENCY` | `USD` | Invoice currency |
| `INVOICE_GENERATION_HOUR` | `0` | UTC hour to generate invoices (0 = midnight) |

## See Also

- [docs/events.md](../../docs/events.md) — event topic definitions
- [docs/deployment.md](../../docs/deployment.md) — change freeze window (critical for billing service)
