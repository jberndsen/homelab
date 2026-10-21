# Next steps

1. Add a finance namespace with [Budgero](https://budgero.app/docs/self-hosting-guide)
   to try replacing YNAB.
2. Add a photos namespace with [Immich](https://docs.immich.app/install/kubernetes/)
   for iPhone photo backup.
3. Plan how to reinstall the second node with Proxmox and use it for cluster
   capacity or backup.
4. Reinstall HAOS, restore solar monitoring and document repeatable setup.
   Configure static IPs for connected devices, including the Envoy and P1 meter.
5. Consider free security scanning alongside Renovate:
   - In `homelab`, run Trivy on PRs and daily: scan
     rendered production Kubernetes configuration and its exact pinned images.
     Publish findings in GitHub; start with reporting before adding merge gates.
   - In future app repos, enable CodeQL, Dependabot alerts, secret scanning and
     Trivy scans of built images, with Renovate handling dependency update PRs.
   - Consider Trivy Operator later for deployed-workload coverage, with its
     configuration in this repo and a way to surface reports. Track host security
     separately; keep updates reviewed and preserve stateful backup requirements.
6. Set up branch protection for `main`: require all changes to go through pull
   requests and prevent direct pushes, including administrator bypasses.
7. Explore proxy-level SSO using Keycloak OIDC, with Envoy replacing Traefik.
8. Resolve the [known Proxmox NUT issue](nut-update.md) through a compatible
   official Debian fix; UPS communication loss and automatic power recovery
   also remain unverified. Existing shutdown protection stays enabled.

## Optional monitoring and recovery follow-ups

The [initial monitoring plan](monitoring-plan.md) is complete.

- Observe retention and disk growth over a full week and measure peak usage.
- Plan a backup restore test, including application consistency and independent
  recovery sources; nightly full-VM backups currently remain untested.
- Add Proxmox, NAS or UPS metrics after confirming device interfaces and access.
- Design notifications, including an external observer for host/VM outages.
- Decide separately whether to retain Homepage. OpenTelemetry is outside the
  completed monitoring scope.
