# Authentication and Authorization

## Identity Provider

The current identity provider is **VaultGuard**.

ARCP migrated from Auth0 to VaultGuard in January 2026 (see [ADR-003](decisions/003-auth-provider.md)). The Auth0 tenant (`acme-robotics.us.auth0.com`) is frozen — it still serves existing sessions during the cutover grace period but no new users should be created there. All new configuration should target VaultGuard.

VaultGuard issuer URL: `https://auth.vaultguard.io/acme-robotics`

## Authentication Flows

### Human operators (OAuth2 + PKCE)

Browser-based operators authenticate via VaultGuard using the Authorization Code flow with PKCE. The auth service acts as the OAuth2 client and issues its own short-lived JWTs after validating the VaultGuard token. This keeps the auth service as the single source of truth for session management.

### Robot devices (mTLS)

Robots do **not** use OAuth2. Each physical robot has a device certificate issued by Acme Robotics' internal Certificate Authority (managed by the auth service). Robots present this certificate for mutual TLS authentication when connecting to the telemetry service's gRPC endpoint.

Device certificates are issued at manufacturing time and renewed annually via the `POST /auth/device/register` endpoint. The auth service validates the robot serial number against the fleet-registry before issuing a certificate.

**Important:** The robot device certificate flow and the human OAuth2 flow are completely separate. Do not attempt to use an OAuth2 access token to authenticate a robot, or vice versa.

## Token Lifetimes

| Token Type | Lifetime | Notes |
|---|---|---|
| Access token (human) | 15 minutes | Validated locally by Kong and services |
| Refresh token (human) | 7 days | Stored in auth service DB; revocable |
| Fleet device certificate | 24 hours | Used for mTLS; renewed automatically by robot firmware |

> **Warning:** The archived [auth0-migration.md](archived/auth0-migration.md) document lists different token lifetimes (60-minute access tokens, 90-day refresh tokens). Those were Auth0-era values and are **no longer valid**. Do not use them.

## JWT Validation

Services must validate JWTs **locally** using the JWKS endpoint published by the auth service:

```
GET http://auth.arcp.svc.cluster.local/auth/.well-known/jwks.json
```

The auth service fetches this from VaultGuard and caches it (TTL: 5 minutes, with stale-while-revalidate to prevent thundering herd on VaultGuard).

**Do not call VaultGuard directly to validate tokens on every request.** In October 2025 (pre-VaultGuard migration, but the lesson applies), the team experienced a production incident where calling the identity provider inline on every API request caused cascading 503s when the provider had a brief slowdown. Kong and all backend services must cache the JWKS locally and validate JWTs using that cached material.

JWKS refresh interval is configured via the `JWKS_REFRESH_INTERVAL` environment variable in the auth service (default: `5m`).

## MFA

Multi-factor authentication is mandatory for all human operators with `PLATFORM_ADMIN` or `TENANT_ADMIN` roles. It is optional but encouraged for `OPERATOR` and `READ_ONLY` roles. MFA is enforced at the VaultGuard level and cannot be bypassed in the auth service.

## Role Model

| Role | Description |
|---|---|
| `PLATFORM_ADMIN` | Acme Robotics internal. Full access to all tenants. |
| `TENANT_ADMIN` | Customer admin. Manages their own tenant's users and fleet. |
| `OPERATOR` | Customer operator. Can create and manage tasks; cannot manage users. |
| `READ_ONLY` | Visibility only. Cannot create tasks or modify fleet config. |
| `ROBOT_DEVICE` | Assigned to mTLS device certificates. Very limited scope — telemetry ingestion only. |

Role claims are embedded in the JWT under the `arcp_roles` claim as a string array.

## See Also

- [services/auth/README.md](../services/auth/README.md) — auth service implementation details
- [decisions/003-auth-provider.md](decisions/003-auth-provider.md) — why VaultGuard was chosen
- [docs/archived/auth0-migration.md](archived/auth0-migration.md) — historical Auth0 configuration (do not use for new implementations)
