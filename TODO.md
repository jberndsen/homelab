# Next steps

1. Back up the YAML repository from the dev laptop to a remote Git repository. Document recovery steps and what is outside the VM backup, including NAS-mounted media.
2. Add Sealed Secrets for version-controlled encrypted secret manifests, with a separate backup of the sealing keys.
3. Add Argo CD to manage deployment reconciliation, adopting existing workloads incrementally.
4. Automate image updating using Renovate.
5. Add cluster and workload observability. Decide separately whether to keep Homepage.
6. Implement a safe image-upgrade workflow using Renovate PRs without automatic merging. Retain release review, application-native pre-upgrade backup, one pinned digest update at a time, rollout verification, and application smoke tests.
7. Add a finance namespace with Budgero to try out replacing YNAB. https://budgero.app/docs/self-hosting-guide
8. Add a photos namespace with a full Immich deployment for iPhone photo backup. https://docs.immich.app/install/docker-compose/ or https://docs.immich.app/install/kubernetes/
9. Brainstorm how I can reinstall my second node with proxmox and make it part of the cluster and either serve as a backup or run some load to contribute.
10. Reinstall HAOS and set it up correctly, currently solar monitoring does not work. Also make sure system documentation for that is good and repeatable. Configure static IPs for everything it connects to (like the solar panel Envoy, P1 meter and so on)