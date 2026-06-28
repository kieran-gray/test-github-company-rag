# Notifications Service

## Responsibilities

The notifications service is a stateless, event-driven fan-out layer that translates ARCP platform events into human-readable communications. It does **not** own any customer data — all information it uses is received via Kafka events at the time of delivery.

Supported channels:
- **Email** — via SendGrid API
- **SMS** — via Twilio API
- **In-app push notifications** — via WebSocket connections to the React dashboard

## Technology

- **Language:** Go 1.22
- **Cache:** Redis — tracks active WebSocket connections for push notifications (key: `ws:user:{user_id}:conn`); enforces per-tenant email rate limiting
- **No persistent database** — the service is intentionally stateless. It does not store notification history; that is the responsibility of the audit log in the auth service.

## Events Consumed

| Topic | Channel(s) Triggered | Notification Content |
|---|---|---|
| `acme.billing.invoice.created` | Email | Invoice ready; includes amount and PDF download link |
| `acme.billing.payment.failed` | Email, SMS | Payment failure alert; prompts to update payment method |
| `acme.auth.user.created` | Email | Welcome email with onboarding instructions |
| `acme.telemetry.robot.alert` (LOW_BATTERY) | In-app push, Email (if critical) | Robot name, battery level, current location |
| `acme.telemetry.robot.alert` (ROBOT_OFFLINE) | In-app push, Email | Robot name, last seen timestamp, last known location |
| `acme.telemetry.robot.alert` (ERROR_CODE) | In-app push | Robot name, error code, description |
| `acme.dispatch.task.completed` | In-app push | Task ID, robot assigned, completion time |
| `acme.fleet.robot.decommissioned` | Email | Robot name and serial number, effective date |

## Events Published

None. The notifications service is a terminal consumer — it never publishes to Kafka.

## Rate Limiting

Email sending is rate-limited to **100 emails per hour per tenant** to prevent alert storms from overwhelming operators during a fleet-wide incident. This limit is enforced using a Redis sliding window counter (`rate:email:{tenant_id}`).

SMS is limited to **20 SMS per hour per tenant** (Twilio cost control).

WebSocket push notifications are not rate-limited — they are low-cost and expected to be high-frequency during active operations.

## Configuration

| Variable | Description |
|---|---|
| `SENDGRID_API_KEY` | SendGrid API key (from GCP Secret Manager) |
| `TWILIO_ACCOUNT_SID` | Twilio account SID |
| `TWILIO_AUTH_TOKEN` | Twilio auth token (from GCP Secret Manager) |
| `EMAIL_FROM_ADDRESS` | Sender address (default: `noreply@arcp.acme.io`) |
| `EMAIL_RATE_LIMIT_PER_HOUR` | Per-tenant hourly email limit (default: 100) |

## Template Management

Email templates are managed in SendGrid and referenced by template ID in the service configuration. Do not hardcode email HTML in the service code. Template IDs are stored in the `helm/notifications/values.yaml` file.

## WebSocket Connection Tracking

When a dashboard user establishes a WebSocket connection, the notifications service stores the connection metadata in Redis. On incoming push-triggering events, the service looks up all active connections for the relevant tenant and delivers the notification. If the user has no active WebSocket connection, push notifications are silently dropped (the user will see missed notifications when they next open the dashboard via in-app notification history).
