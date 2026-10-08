# Next steps

TODO 1–3 are complete under [the staged plan](argocd-plan.md): checkpoints
1–4 and Execution 6's documentation audit passed. This finishes the Argo rollout;
automatic reconciliation remains enabled. Recovery limits are recorded in the
[operations guide](infrastructure/node-main/ns-argo/SETUP.md#choose-the-recovery-path).

1. [x] Repository/recovery documentation: public GitHub source, fresh-cluster and ordinary/guarded VM recovery, and separate NAS/repository/key recovery needs documented and safely checked.
2. [x] Encrypted Secrets: independent encrypted key recovery and both production adoptions verified; final read-only checks confirmed unchanged credentials and key-backup coverage.
3. [x] Argo CD: automatic sync, self-healing and pruning demonstrated; operating/recovery documentation audited, with storage safeguards preserved.
4. Automate image updating using Renovate.
5. Add cluster and workload observability. Decide separately whether to keep Homepage.
6. Implement a safe image-upgrade workflow using Renovate PRs without automatic merging. Retain release review, application-native pre-upgrade backup, one pinned digest update at a time, rollout verification, and application smoke tests.
7. Add a finance namespace with Budgero to try out replacing YNAB. https://budgero.app/docs/self-hosting-guide
8. Add a photos namespace with a full Immich deployment for iPhone photo backup. https://docs.immich.app/install/docker-compose/ or https://docs.immich.app/install/kubernetes/
9. Brainstorm how I can reinstall my second node with proxmox and make it part of the cluster and either serve as a backup or run some load to contribute.
10. Reinstall HAOS and set it up correctly, currently solar monitoring does not work. Also make sure system documentation for that is good and repeatable. Configure static IPs for everything it connects to (like the solar panel Envoy, P1 meter and so on)
