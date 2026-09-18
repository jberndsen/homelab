# Next steps

1. Enable NUT server on the UPS and send shutdown signals to vm-ubuntu-k3s.
2. Back up the YAML repository from the dev laptop to a remote Git repository. Document recovery steps and what is outside the VM backup, including NAS-mounted media.
3. Add Sealed Secrets for version-controlled encrypted secret manifests, with a separate backup of the sealing keys.
4. Add Argo CD to manage deployment reconciliation, adopting existing workloads incrementally.
5. Add cluster and workload observability. Decide separately whether to keep Homepage.
6. Implement a safe image-upgrade workflow using Renovate PRs without automatic merging. Retain release review, application-native pre-upgrade backup, one pinned digest update at a time, rollout verification, and application smoke tests.
7. Add a finance namespace with Budgero to try out replacing YNAB.
8. Add a photos namespace with a full Immich deployment for iPhone photo backup.
