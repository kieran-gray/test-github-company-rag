# ADR-003: Migrate Identity Provider from Auth0 to VaultGuard

**Status:** Accepted (2025-09-01) — Migration completed 2026-01-15
**Date:** 2025-09-01
**Decision makers:** Sarah Chen (Platform Architect), James Okafor (Security Lead), Priya Nair (VP Engineering)

---

## Context

ARCP has used Auth0 as its identity provider since the v2.0.0 launch in June 2024. In July 2025, Auth0 (Okta) announced a pricing restructure that increased Acme Robotics' annual identity costs by approximately 340%, from $28,000/year to $95,000/year, driven primarily by the per-MAU pricing model applied to robot device identities.

Additionally, as robot fleet sizes grew, two technical limitations of Auth0 became significant:

1. **No native mTLS device authentication.** Robots need to authenticate using device certificates (mutual TLS) at the gRPC transport layer. Auth0's M2M tokens are JWT-based and were being misused as a workaround — robots would fetch a token via client_credentials grant and embed it in gRPC metadata. This was fragile and did not provide the certificate-level identity guarantee the security team required.

2. **EU data residency.** Acme Robotics signed Müller Logistics GmbH (a major German customer) in August 2025. Müller's data processing agreement requires that EU operator identity data be stored and processed in the EU. Auth0's EU deployment had limitations that made compliance difficult to achieve.

## Decision

Migrate from Auth0 to VaultGuard.

VaultGuard was selected from a shortlist of four alternatives (Auth0 retained, Keycloak self-hosted, Okta Customer Identity, VaultGuard) based on the following evaluation:

| Criterion | Auth0 (retained) | Keycloak (self-hosted) | Okta Customer Identity | VaultGuard |
|---|---|---|---|---|
| Annual cost | $95k | ~$15k (hosting) | $78k | $34k |
| Native mTLS device auth | No | Via plugins | No | Yes (first-class) |
| EU data residency | Limited | Full (self-hosted) | Yes | Yes |
| Team familiarity | High | Low | Medium | Low |
| Audit log quality | Medium | High | Medium | High |

VaultGuard provided the best balance: 64% lower cost than retained Auth0, native mTLS device certificate issuance and validation, EU tenant data residency in Frankfurt region, and strong audit log capabilities required by Acme Robotics' internal security policy.

The cost of switching (team learning, migration effort) was estimated at 6 engineer-weeks, acceptable given the $61k/year ongoing savings.

## Migration Plan (Executed)

The migration was executed in two phases:

**Phase 1 — Human users (November 2025):**
- VaultGuard tenant provisioned in EU region
- All existing Auth0 user accounts migrated via VaultGuard's Auth0 import tool
- Auth service updated to validate tokens from VaultGuard issuer
- Parallel validation period: both Auth0 and VaultGuard tokens accepted for 3 weeks
- Auth0 human user logins disabled 2025-11-30

**Phase 2 — Robot device authentication (January 2026):**
- Auth service extended with `POST /auth/device/register` endpoint backed by VaultGuard CA
- Robot firmware updated (v3.1.0 OTA update) to use new mTLS certificate flow
- Legacy M2M token flow disabled 2026-01-15

**Migration completed:** 2026-01-15. Both phases are now complete.

## Current State

VaultGuard is the sole identity provider. The Auth0 tenant (`acme-robotics.us.auth0.com`) remains live but is frozen — no new users can be created. It will be decommissioned fully in Q3 2026 after the 6-month retention period for audit logs expires.

Historical Auth0 configuration details are in [docs/archived/auth0-migration.md](../archived/auth0-migration.md). **Do not use those values for new implementations.**

## Token Lifetime Changes

As part of the migration, token lifetimes were updated to align with VaultGuard recommendations and Acme Robotics security policy:

| Token | Auth0 era | VaultGuard era |
|---|---|---|
| Access token | 60 minutes | 15 minutes |
| Refresh token | 90 days | 7 days |
| Device certificate | N/A (M2M tokens, 24h) | 24 hours (mTLS cert) |
