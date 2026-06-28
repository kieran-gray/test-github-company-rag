## Summary

_Describe what this PR does and why. Link to any relevant issue, ADR, or Slack discussion._

---

## Type of Change

- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (API or event schema change that requires coordinated deploy or consumer updates)
- [ ] Architecture Decision Record (new ADR in `docs/decisions/`)
- [ ] Documentation only
- [ ] Refactor / chore (no behavior change)

---

## Affected Services

_Check all services whose behavior, events, or APIs are modified by this PR._

- [ ] auth
- [ ] fleet-registry
- [ ] dispatch
- [ ] telemetry
- [ ] billing
- [ ] notifications
- [ ] Shared infrastructure (Helm, Terraform, CI/CD)

---

## Checklist

- [ ] Tests added or updated (unit and/or integration)
- [ ] `docker compose -f docker-compose.test.yml up --abort-on-container-exit` passes locally
- [ ] Documentation updated (service README, `docs/events.md` if topics changed, `docs/architecture.md` if architecture changed)
- [ ] CHANGELOG.md updated under `[Unreleased]`
- [ ] ADR created if this PR introduces a significant architectural decision
- [ ] No breaking changes to Kafka event schemas (additive only — or schema version bumped and consumers updated)

---

## Deployment Notes

_Does this require a coordinated deploy with another service? Are there database migrations? Is there a rollback risk?_

- Requires coordinated deploy with: _(list services or "none")_
- Database migrations: _(yes/no — if yes, list migration files)_
- Rollback: _(describe any special rollback considerations, or "standard Helm rollback")_
- Change freeze: _(does this need to land before or after the billing cycle close? See [docs/deployment.md](docs/deployment.md))_

---

## Security Considerations

- [ ] This PR **does not** affect authentication or authorization logic
- [ ] This PR **does not** change Kafka topic ACLs or event schema consumer access
- [ ] This PR **does not** affect tenant isolation (no changes to ORM tenant_id enforcement)
- [ ] This PR **does not** introduce new external dependencies (or security review requested in `#security`)

_If any box above is unchecked, describe the security impact and confirm review with James Okafor (Security Lead)._

---

## Testing Done

_Describe how you tested this change: unit tests, integration tests, manual testing in local docker compose stack, or staging._
