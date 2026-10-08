# Next steps

TODO 1–3 are in progress under [the staged plan](argocd-plan.md). Checkpoints
1–2 are verified and pushed; paused before checkpoint 3. Leave these open until
media adoption, reconciliation demonstrations and final recovery documentation
have passed all planned checks.

1. Finish repository/recovery documentation. Public GitHub remote is established; document and validate all recovery paths, including NAS media outside the VM backup.
2. Finish encrypted Secret adoption. Sealed Secrets controller and independent encrypted key recovery are verified; seal/adopt the two production Secrets during checkpoint 3.
3. Finish Argo CD adoption and reconciliation. Bootstrap and manual-sync Applications are verified; media adoption is checkpoint 3 and automatic reconciliation is checkpoint 4.
4. Automate image updating using Renovate.
5. Add cluster and workload observability. Decide separately whether to keep Homepage.
6. Implement a safe image-upgrade workflow using Renovate PRs without automatic merging. Retain release review, application-native pre-upgrade backup, one pinned digest update at a time, rollout verification, and application smoke tests.
7. Add a finance namespace with Budgero to try out replacing YNAB. https://budgero.app/docs/self-hosting-guide
8. Add a photos namespace with a full Immich deployment for iPhone photo backup. https://docs.immich.app/install/docker-compose/ or https://docs.immich.app/install/kubernetes/
9. Brainstorm how I can reinstall my second node with proxmox and make it part of the cluster and either serve as a backup or run some load to contribute.
10. Reinstall HAOS and set it up correctly, currently solar monitoring does not work. Also make sure system documentation for that is good and repeatable. Configure static IPs for everything it connects to (like the solar panel Envoy, P1 meter and so on)
