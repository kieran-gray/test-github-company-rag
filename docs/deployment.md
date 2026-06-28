# Deployment

## Overview

ARCP services are deployed to Google Kubernetes Engine (GKE) via GitHub Actions CI/CD. Each service is packaged as a Helm chart (located in `helm/{service-name}/`). There are three environments: dev, staging, and production.

## Environments

| Environment | Cluster | Auto-deploy trigger | URL |
|---|---|---|---|
| dev | `gke-arcp-dev` | Merge to `main` branch | `https://dev.arcp-internal.acme.io` |
| staging | `gke-arcp-staging` | Dev smoke tests pass | `https://staging.arcp-internal.acme.io` |
| production | `gke-arcp-prod` | Manual trigger (with approvals) | `https://app.arcp.acme.io` |

## CI/CD Pipeline

The GitHub Actions pipeline (`.github/workflows/deploy.yml`) runs the following stages in order:

1. **Build** — Docker images built and pushed to Google Artifact Registry (`us-central1-docker.pkg.dev/acme-robotics/arcp/`)
2. **Test** — Unit tests and integration tests run against a local Docker Compose stack
3. **Scan** — Container images scanned with Trivy for known CVEs; pipeline fails on CRITICAL findings
4. **Deploy to dev** — Helm upgrade applied to dev cluster on successful test pass
5. **Smoke tests** — Automated smoke test suite runs against dev (robot registration, task dispatch, telemetry ingestion)
6. **Deploy to staging** — Triggered automatically when dev smoke tests pass
7. **Staging verification** — 15-minute bake window with automated monitoring checks
8. **Deploy to production** — Manual trigger only; see checklist below

## Production Deploy Checklist

A production deployment requires all of the following:

- [ ] All CI pipeline stages passed (build, test, scan, dev, staging)
- [ ] Staging smoke tests green for at least 30 minutes
- [ ] Two approvals in the GitHub Actions deployment gate — **one must be from the `platform-leads` GitHub team**
- [ ] Change freeze window check passed (see below)
- [ ] Rollback procedure confirmed (Helm revision noted)

To trigger a production deploy: go to the GitHub Actions tab, select the `Deploy to Production` workflow, and click "Run workflow". Select the image tag (typically the same tag promoted through dev → staging).

## Change Freeze

Production deployments are frozen during the **last 3 business days of each calendar month**. This is the billing cycle close period — the billing service is running monthly invoice generation and payment processing jobs. Deploying during this window risks interrupting those jobs.

Non-critical deployments should be scheduled before or after the freeze. Emergency hotfixes can be deployed with VP Engineering approval (Priya Nair or delegate).

## Canary Deployments

The **dispatch** and **telemetry** services use canary deployments because they handle the highest traffic volumes and a bad deploy can affect the entire robot fleet.

Canary configuration (in Helm values):
- Initial traffic split: 10% to canary, 90% to stable
- Bake time: 30 minutes at 10% before promotion
- Promotion: automatic if error rate < 0.5% and p99 latency < 200ms; otherwise rollback

All other services use a standard rolling update strategy.

## Rollback

To roll back a service to the previous Helm release:

```bash
# List revisions for a service
helm history dispatch -n arcp-prod

# Roll back to the previous revision
helm rollback dispatch -n arcp-prod

# Roll back to a specific revision number
helm rollback dispatch 14 -n arcp-prod
```

Helm rollbacks take effect immediately. Monitor the `#platform-alerts` Slack channel for error rate after rollback.

## Configuration Management

Service configuration is managed via Kubernetes Secrets, which are synchronized from GCP Secret Manager by External Secrets Operator. **Never commit secrets to this repository.** If a secret is accidentally committed, rotate it immediately and notify the security team (`#security` on Slack).

To add a new configuration value:
1. Add it to GCP Secret Manager (project `acme-robotics-prod`)
2. Add a reference in `helm/{service}/values.yaml` under `externalSecrets`
3. Reference the Kubernetes Secret in your service's Deployment manifest

## Historical Note

Prior to v3.0.0 (June 2025), ARCP was deployed manually via SSH to Google Compute Engine instances. That process is documented in [docs/archived/old-deployment.md](archived/old-deployment.md) for historical reference only. Do not follow those instructions for any current environment.
