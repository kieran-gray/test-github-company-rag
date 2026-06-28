# Auth Service

## Responsibilities

The auth service is the single source of truth for identity in ARCP. It handles:

- **Human operator authentication** via VaultGuard OAuth2 (Authorization Code + PKCE)
- **Session management** — issues short-lived JWTs after validating VaultGuard tokens; manages refresh token lifecycle in PostgreSQL
- **Robot device certificate issuance** — signs mTLS device certificates for robot hardware via VaultGuard's CA integration
- **JWKS endpoint** — proxies and caches VaultGuard's JWKS response so downstream services can validate JWTs locally without calling VaultGuard on every request
- **MFA enforcement** — delegates MFA challenge/response to VaultGuard; enforces that `PLATFORM_ADMIN` and `TENANT_ADMIN` sessions completed MFA before issuing tokens

## Technology

- **Language:** Go 1.22
- **Database:** PostgreSQL (`auth_db`) — stores user accounts, refresh tokens, robot device registrations, audit log
- **Cache:** Redis — JWKS cache (TTL: 5 min, stale-while-revalidate enabled), session revocation list

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/login` | Initiate OAuth2 redirect to VaultGuard |
| `GET` | `/auth/callback` | OAuth2 callback handler; exchanges code for tokens |
| `POST` | `/auth/refresh` | Refresh an access token using a refresh token |
| `POST` | `/auth/logout` | Revoke refresh token and clear session |
| `GET` | `/auth/.well-known/jwks.json` | JWKS endpoint (cached from VaultGuard) |
| `POST` | `/auth/device/register` | Issue a device certificate for a robot (validates serial against fleet-registry) |

## Events Published

| Topic | When |
|---|---|
| `acme.auth.user.created` | A new user account is created |
| `acme.auth.user.deactivated` | A user account is deactivated by a TENANT_ADMIN or PLATFORM_ADMIN |

## Events Consumed

None. Auth is upstream in the dependency graph; no other service's events should affect auth state.

## Configuration

| Variable | Default | Description |
|---|---|---|
| `VAULTGUARD_ISSUER_URL` | — | VaultGuard issuer URL (required) |
| `VAULTGUARD_CLIENT_ID` | — | OAuth2 client ID (required) |
| `VAULTGUARD_CLIENT_SECRET` | — | OAuth2 client secret (required, from GCP Secret Manager) |
| `JWKS_REFRESH_INTERVAL` | `5m` | How often to refresh JWKS from VaultGuard |
| `ACCESS_TOKEN_TTL` | `15m` | Access token lifetime |
| `REFRESH_TOKEN_TTL` | `168h` | Refresh token lifetime (7 days) |
| `DEVICE_CERT_TTL` | `24h` | Robot device certificate lifetime |

## Known Issues

**JWKS refresh and 401 storms:** If VaultGuard is slow to respond to a JWKS refresh request, the auth service may briefly serve stale JWKS. Stale-while-revalidate caching is implemented to mitigate this — the service will serve the stale JWKS while refreshing in the background, rather than blocking requests. However, if VaultGuard is down for longer than the JWKS TTL, downstream services will eventually receive 401s as cached JWKS expire. This scenario is tracked in the incident runbook.

## See Also

- [docs/authentication.md](../../docs/authentication.md) — full auth architecture
- [decisions/003-auth-provider.md](../../docs/decisions/003-auth-provider.md) — why VaultGuard was chosen
