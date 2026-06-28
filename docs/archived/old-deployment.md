# Legacy Deployment Procedure

> **SUPERSEDED — This document applies to ARCP v1.x and v2.x (before June 2025).**
> **Current deployment is via GitHub Actions + Helm. See [docs/deployment.md](../deployment.md).**

---

## Applicability

This procedure was used for all ARCP deployments from launch (v2.0.0, June 2024) through approximately v2.8.0 (August 2025). It describes a manual SSH-based deployment to Google Compute Engine (GCE) instances running Docker containers under systemd.

**Do not follow these steps for any current deployment.** They will not work in the current GKE-based infrastructure.

## Old Infrastructure

| Environment | GCE Instance | Zone |
|---|---|---|
| Production | `arcp-prod-001` (e2-standard-8) | us-central1-a |
| Staging | `arcp-staging-001` (e2-standard-4) | us-central1-b |

Both instances ran Docker with a `docker-compose.prod.yml` file managing all services as a single monolith container plus PostgreSQL, Redis, and RabbitMQ containers.

## Old Deployment Procedure

1. **SSH into the production GCE instance:**
   ```bash
   gcloud compute ssh arcp-prod-001 --zone us-central1-a
   ```

2. **Pull the new Docker image:**
   ```bash
   docker pull us-central1-docker.pkg.dev/acme-robotics/arcp/arcp-monolith:{tag}
   ```

3. **Update the image tag in docker-compose.prod.yml:**
   ```bash
   sed -i 's/arcp-monolith:.*/arcp-monolith:{tag}/' /opt/arcp/docker-compose.prod.yml
   ```

4. **Restart the application container:**
   ```bash
   systemctl restart arcp-app
   ```
   The `arcp-app` systemd unit ran `docker compose -f /opt/arcp/docker-compose.prod.yml up -d arcp-monolith`.

5. **Manual health check:**
   ```bash
   curl -f http://localhost:8080/health
   ```

6. **If health check fails, rollback by reverting the previous image tag** and restarting.

## Limitations of the Old Process

- No automated testing gate before deployment
- No canary — the monolith replaced 100% of traffic immediately
- Rollback was manual and error-prone (required knowing the previous tag)
- No approval workflow — any engineer with GCE SSH access could deploy to production
- Staging and production were on separate instances but the deployment process was identical — no pipeline distinction
- RabbitMQ and PostgreSQL were running on the same GCE instance as the application (no HA, single point of failure)

These limitations were a key driver for the move to GitHub Actions + GKE + Helm (see [docs/deployment.md](../deployment.md)).

## When These Instances Were Decommissioned

The GCE production instance (`arcp-prod-001`) was shut down on **2025-07-01**, after v3.0.0 successfully ran on GKE for 14 days. The staging GCE instance was shut down on **2025-06-15**.

Backups of the PostgreSQL data from the monolith era were migrated to the per-service databases as part of the v3.0.0 migration. Those backups are archived in GCS bucket `gs://acme-robotics-db-archives/monolith-2025-06/`.
