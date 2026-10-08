# Mission: Operate this homelab through Git

## Why
Run the existing K3s media stack through Argo CD while retaining working data,
recoverable secrets and a clear way to rebuild. Learn the concepts through the
actual implementation so everyday changes and recovery are understandable.

## Success looks like
- Explain how Git, an ApplicationSet, Applications and Kubernetes controllers interact.
- Review and deploy changes, pause reconciliation and verify media health.
- Recover configuration, sealing keys and application data from their respective backups.

## Constraints
- Teach alongside small implementation steps and pause at the plan's four checkpoints.
- Preserve existing storage identities, settings, credentials and media image digests.
- Use the confirmed system records; ask the owner about unknown infrastructure facts.

## Out of scope
- Renovate, observability, TLS/SSO and new data backup systems in this rollout.
