# Next steps

1. Add Renovate for reviewed image-update PRs without automatic merging. Retain
   release review, an application-native backup before upgrade, one pinned digest
   update at a time, rollout verification and application smoke tests.
2. Add cluster and workload observability. Decide separately whether to keep Homepage.
3. Add a finance namespace with [Budgero](https://budgero.app/docs/self-hosting-guide)
   to try replacing YNAB.
4. Add a photos namespace with [Immich](https://docs.immich.app/install/kubernetes/)
   for iPhone photo backup.
5. Plan how to reinstall the second node with Proxmox and use it for cluster
   capacity or backup.
6. Reinstall HAOS, restore solar monitoring and document repeatable setup.
   Configure static IPs for connected devices, including the Envoy and P1 meter.
