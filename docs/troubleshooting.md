# Troubleshooting

A guide to the most common operational issues in ARCP and how to diagnose them.

## Login Fails

**Symptoms:** Operators cannot log in; `401 Unauthorized` from auth service; dashboard shows "Authentication failed."

**Diagnosis steps:**

1. **Check VaultGuard health** — go to `https://status.vaultguard.io` and check for incidents affecting the Acme Robotics tenant. VaultGuard outages will prevent all new logins.

2. **Check the auth service logs:**
   ```bash
   kubectl logs -n arcp-prod -l app=auth --tail=100
   ```
   Look for `JWKS refresh failed` or `VaultGuard token validation error`.

3. **Check JWT expiry** — access tokens expire after 15 minutes. If users report intermittent login failures, they may be using an expired refresh token (7-day lifetime). They should log out and log in again.

4. **Check Redis session cache** — if Redis is unavailable, the auth service cannot validate sessions:
   ```bash
   kubectl exec -n arcp-prod deployment/auth -- redis-cli -h redis.internal ping
   ```

5. **Check the JWKS endpoint** — the auth service caches JWKS from VaultGuard. If the cache is stale (e.g. after a key rotation), services may reject valid tokens:
   ```bash
   curl http://auth.arcp.svc.cluster.local/auth/.well-known/jwks.json
   ```
   Force a JWKS refresh by restarting the auth service pod.

> **Historical note:** Do not attempt to validate tokens directly against Auth0 (`acme-robotics.us.auth0.com`). The Auth0 tenant is frozen and no longer the active IDP.

---

## Robot Not Appearing in Dashboard

**Symptoms:** A robot was powered on and is sending heartbeats but does not appear in the fleet dashboard.

**Diagnosis steps:**

1. **Verify robot registration** — check that `acme.fleet.robot.registered` was published. Use the Kafka console consumer:
   ```bash
   kafka-console-consumer.sh --bootstrap-server kafka.internal:9092 \
     --topic acme.fleet.robot.registered --from-beginning | grep <robot_id>
   ```

2. **Check fleet-registry consumer lag** — if the fleet-registry consumer is lagging, registration events may not be processed yet:
   ```bash
   kafka-consumer-groups.sh --bootstrap-server kafka.internal:9092 \
     --describe --group arcp-fleet-registry
   ```

3. **Check Redis robot state cache** — the dashboard reads robot state from Redis. If the robot sent heartbeats but its state key isn't in Redis, check the telemetry service:
   ```bash
   kubectl exec -n arcp-prod deployment/telemetry -- \
     redis-cli -h redis.internal get robot:<robot_id>:state
   ```

4. **Check telemetry service logs** for gRPC authentication errors — the robot may be failing mTLS handshake due to an expired device certificate.

---

## Tasks Not Being Dispatched

**Symptoms:** Tasks are created via the API but robots are not picking them up; tasks stuck in `PENDING` state.

**Diagnosis steps:**

1. **Check dispatch service logs:**
   ```bash
   kubectl logs -n arcp-prod -l app=dispatch --tail=200
   ```
   Look for `no available robots` or `robot state cache miss`.

2. **Check Kafka consumer group lag** for `acme.dispatch.task.assigned`:
   ```bash
   kafka-consumer-groups.sh --bootstrap-server kafka.internal:9092 \
     --describe --group arcp-dispatch
   ```

3. **Verify robot availability** — dispatch only assigns tasks to robots in `AVAILABLE` state. Check Redis:
   ```bash
   kubectl exec -n arcp-prod deployment/dispatch -- \
     redis-cli -h redis.internal keys "robot:*:state"
   ```
   If no robots are AVAILABLE, check if recent telemetry alerts moved them to UNAVAILABLE.

4. **Check for active CRITICAL telemetry alerts** — a recent `acme.telemetry.robot.alert` event with severity CRITICAL may have moved all robots to UNAVAILABLE. Check the DLQ topic if alerts are not processing:
   ```bash
   kafka-console-consumer.sh --topic acme.dlq.acme.telemetry.robot.alert \
     --bootstrap-server kafka.internal:9092 --from-beginning
   ```

---

## Events Not Processing

**Symptoms:** Services appear healthy but events are not flowing; downstream effects (billing, notifications) not happening.

**Diagnosis steps:**

1. **Check Kafka broker health:**
   ```bash
   kafka-topics.sh --bootstrap-server kafka.internal:9092 --list
   ```
   If this times out, the Kafka brokers may be down. Check the `#platform-alerts` Slack channel.

2. **Check for dead letter queue messages:**
   ```bash
   kafka-console-consumer.sh --bootstrap-server kafka.internal:9092 \
     --topic acme.dlq.acme.dispatch.task.assigned --from-beginning
   ```
   Messages in the DLQ indicate consumer failures. Check the consuming service's logs for the error.

3. **Check Schema Registry** — a schema incompatibility can cause consumers to fail deserialization:
   ```bash
   curl http://schema-registry.internal/subjects
   ```

4. **Check consumer group lag across all groups:**
   ```bash
   kafka-consumer-groups.sh --bootstrap-server kafka.internal:9092 --list | \
     xargs -I{} kafka-consumer-groups.sh --bootstrap-server kafka.internal:9092 \
     --describe --group {}
   ```

---

## Slow APIs

**Symptoms:** API response times are elevated; users report dashboard slowness.

**Diagnosis steps:**

1. **PostgreSQL connection pool exhaustion** — check `pg_stat_activity` on the relevant service's database:
   ```sql
   SELECT count(*), state FROM pg_stat_activity GROUP BY state;
   ```
   If most connections are `idle in transaction`, a service may be holding transactions open too long.

2. **Redis cache hit rate** — if Redis hit rate is low, services are falling through to PostgreSQL for every request. Check Redis INFO:
   ```bash
   redis-cli -h redis.internal INFO stats | grep keyspace
   ```

3. **Kong rate limiting** — if a tenant is hitting their rate limit (1,000 req/min default), they will receive `429 Too Many Requests`. Check Kong admin API for rate limit status.

4. **TimescaleDB query performance** (telemetry service only) — run `EXPLAIN ANALYZE` on slow historical queries to check if the hypertable chunk exclusion is working correctly.

---

## Billing Discrepancies

**Symptoms:** A tenant reports their invoice doesn't match the number of active robots or tasks.

**Diagnosis steps:**

1. **Audit billing events in Kafka** — the billing service generates invoices based on `acme.fleet.robot.registered` and `acme.fleet.robot.decommissioned` events. Replay the topic for the billing period in question:
   ```bash
   kafka-console-consumer.sh --topic acme.fleet.robot.registered \
     --bootstrap-server kafka.internal:9092 --from-beginning | \
     grep <tenant_id>
   ```

2. **Cross-reference with dispatch completions** — billing also meters on `acme.dispatch.task.completed`. Confirm task count matches the invoice line items.

3. **Check for duplicate event processing** — if the billing service processed an event twice (consumer restart during rebalance), it may have double-charged. Check `event_id` deduplication records in the billing DB.
