# Contributing to ARCP

This document describes the conventions and requirements for contributing to the Acme Robotics Control Platform codebase.

## Branch Naming

All branches must follow this naming convention:

| Prefix | When to use | Example |
|---|---|---|
| `feature/` | New functionality | `feature/ota-update-support` |
| `bugfix/` | Bug fixes for non-production issues | `bugfix/dispatch-assignment-timeout` |
| `hotfix/` | Urgent production fixes | `hotfix/billing-double-charge` |
| `adr/` | Architecture Decision Records | `adr/005-add-grpc-gateway` |
| `chore/` | Maintenance tasks with no behavior change | `chore/update-go-dependencies` |

Branch names should be lowercase, hyphenated, and descriptive enough to understand without reading the commits.

## Commit Messages

ARCP follows [Conventional Commits](https://www.conventionalcommits.org/). Commits that do not pass the commit-msg hook will be rejected.

Format: `type(scope): description`

### Types

- `feat` — new feature
- `fix` — bug fix
- `docs` — documentation only
- `refactor` — code change that neither fixes a bug nor adds a feature
- `test` — adding or updating tests
- `chore` — dependency updates, build changes, CI config
- `perf` — performance improvement
- `adr` — adding an Architecture Decision Record

### Scopes

Use the service or domain the change applies to:

| Scope | When to use |
|---|---|
| `fleet` | fleet-registry service |
| `telemetry` | telemetry service |
| `auth` | auth service |
| `billing` | billing service |
| `dispatch` | dispatch service |
| `notifications` | notifications service |
| `events` | Kafka topics, schemas, or event contracts |
| `docs` | Top-level documentation changes |
| `infra` | Helm charts, Terraform, CI/CD pipeline |
| `deps` | Dependency updates (Go modules, Docker base images) |

### Examples

```
feat(dispatch): add priority queue with four-tier classification
fix(billing): prevent double-counting robots in MAINTENANCE state
docs(events): add acme.billing.payment.failed to topic catalogue
adr(auth): document VaultGuard migration decision
chore(deps): upgrade Go to 1.22.4
```

## Pull Requests

Every pull request must have:

- [ ] **Tests added or updated** — unit tests for new logic, integration tests for new event flows or API endpoints. Run tests with `docker compose -f docker-compose.test.yml up --abort-on-container-exit`
- [ ] **Documentation updated** — if you change an API endpoint, update the service README. If you change a Kafka topic, update [docs/events.md](docs/events.md). If you make an architectural decision, create an ADR.
- [ ] **Two reviewers** — at least one reviewer must be from the `platform-team` GitHub group. The second reviewer can be any engineer familiar with the affected service.
- [ ] **Changelog entry** — add an entry to CHANGELOG.md under the appropriate version (create an `## [Unreleased]` section if one doesn't exist)
- [ ] **ADR created if applicable** — if your PR introduces a significant architectural decision (new dependency, new communication pattern, change to auth model), create a corresponding ADR in `docs/decisions/`

## Service Ownership

| Service | Primary Owner | Slack Channel |
|---|---|---|
| auth | James Okafor | `#service-auth` |
| fleet-registry | Marcus Webb | `#service-fleet` |
| dispatch | Sarah Chen | `#service-dispatch` |
| telemetry | David Park | `#service-telemetry` |
| billing | Yuna Kim | `#service-billing` |
| notifications | Marcus Webb | `#service-notifications` |

If your PR touches a service, add the primary owner as a required reviewer.

## Running Tests

### Unit tests

```bash
# Run unit tests for a specific service
go test ./services/dispatch/...

# Run all unit tests
go test ./...
```

### Integration tests

Integration tests spin up a full Docker stack (Kafka, PostgreSQL, TimescaleDB, Redis) and run against it:

```bash
docker compose -f docker-compose.test.yml up --abort-on-container-exit
```

Integration tests are required for:
- New Kafka producer/consumer paths
- New database migrations
- New API endpoints with auth middleware

### Pre-commit checks

The repository uses `pre-commit` hooks for:
- `gofmt` / `goimports` formatting
- `golangci-lint` with the project's `.golangci.yml` config
- `conventional-commit` message format validation

Install hooks with: `pre-commit install`

## ADR Template

When creating a new Architecture Decision Record in `docs/decisions/`:

```markdown
# ADR-XXX: Title

**Status:** Proposed / Accepted / Superseded by ADR-YYY
**Date:** YYYY-MM-DD
**Decision makers:** Name (Role), Name (Role)

---

## Context

What is the problem or opportunity this decision addresses?

## Decision

What was decided?

## Alternatives Considered

What else was evaluated, and why was it rejected?

## Consequences

What are the positive and negative effects of this decision?
```

Number ADRs sequentially. Do not reuse numbers, even if an ADR is rejected.
