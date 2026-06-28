# Auth0 Migration Notes

> **ARCHIVED — This document describes the migration away from Auth0, which was completed in January 2026.**
> **The information below reflects historical Auth0 configuration. Do NOT use these values for new implementations.**
> **Current authentication configuration is in [docs/authentication.md](../authentication.md).**

---

## Background

ARCP used Auth0 (Okta) as its identity provider from launch (v2.0.0, June 2024) through January 2026. This document captures the Auth0 configuration and the migration steps followed during the transition to VaultGuard. It is retained for audit purposes.

## Historical Auth0 Configuration

**Auth0 tenant:** `acme-robotics.us.auth0.com`
**Region:** US East (us-east-1)

### Token Lifetimes (Auth0 era — DO NOT USE)

| Token | Lifetime |
|---|---|
| Access token | **60 minutes** |
| Refresh token | **90 days** |

> ⚠️ **Warning:** These values are significantly longer than the current VaultGuard configuration (15-minute access tokens, 7-day refresh tokens). If you are referencing this document for token lifetime configuration, stop — use [docs/authentication.md](../authentication.md) instead.

### Auth0 Application Configuration

- **Application type:** Regular Web Application (for dashboard) + Machine-to-Machine (for robot workaround)
- **M2M token approach:** Robot authentication in the Auth0 era used client_credentials grant with a per-robot M2M application. Each robot was registered as a separate Auth0 application — this became unmanageable at fleet sizes above ~200 robots and was one of the drivers for migrating to VaultGuard's native mTLS device auth.
- **MFA:** Enforced for admin users only (TENANT_ADMIN and above). MFA was not enforced for OPERATOR role due to Auth0 pricing tier limitations.

### Auth0 Roles

The Auth0 role mapping was:
- `arcp:platform_admin`
- `arcp:tenant_admin`
- `arcp:operator`
- `arcp:read_only`

These are now mapped to the `arcp_roles` JWT claim using the same names under VaultGuard.

## Migration Steps Followed

1. VaultGuard tenant provisioned: `acme-robotics` (EU Frankfurt region)
2. All Auth0 users exported and imported into VaultGuard via the Auth0 import connector
3. Auth service updated to accept tokens from both Auth0 and VaultGuard issuers (dual-issuer mode, November 2025)
4. Dashboard updated to redirect login flow to VaultGuard
5. Monitoring deployed to track Auth0 vs VaultGuard login counts
6. Auth0 human logins disabled after VaultGuard traffic reached 95% (2025-11-30)
7. Robot M2M tokens migrated to mTLS device certs via v3.1.0 OTA firmware update (2026-01)
8. Auth0 tenant frozen (2026-01-15)

## Current Auth0 Status

The Auth0 tenant is **frozen**. It is retained for:
- Audit log access (required for 12 months post-migration per Acme Robotics security policy)
- Emergency rollback option (not expected to be used)

The tenant will be fully decommissioned in **Q3 2026**. Do not add new users, applications, or rules to the Auth0 tenant.

## See Also

- [decisions/003-auth-provider.md](../decisions/003-auth-provider.md) — rationale for the migration
- [docs/authentication.md](../authentication.md) — current VaultGuard configuration
