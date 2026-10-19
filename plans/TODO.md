# Next steps

1. Finish [the monitoring lessons](monitoring-plan.md). Prometheus, VM metrics,
   Kubernetes state/container metrics and Grafana are deployed. Remaining:
   - Lesson 4: verify VM network collection, build the learning dashboard and
     pass the checkpoint. Stopped for the evening before the first saved panel.
   - Lesson 5: dashboard provisioning, persistence tests, resource/retention
     checks and recovery notes. Not started.
   - Optional later: Proxmox/NAS/UPS metrics and notifications. Decide separately
     whether to keep Homepage; OpenTelemetry is outside the current scope.
2. Add a finance namespace with [Budgero](https://budgero.app/docs/self-hosting-guide)
   to try replacing YNAB.
3. Add a photos namespace with [Immich](https://docs.immich.app/install/kubernetes/)
   for iPhone photo backup.
4. Plan how to reinstall the second node with Proxmox and use it for cluster
   capacity or backup.
5. Reinstall HAOS, restore solar monitoring and document repeatable setup.
   Configure static IPs for connected devices, including the Envoy and P1 meter.
6. Consider free security scanning alongside Renovate:
   - In `homelab`, run Trivy on PRs and daily: scan
     rendered production Kubernetes configuration and its exact pinned images.
     Publish findings in GitHub; start with reporting before adding merge gates.
   - In future app repos, enable CodeQL, Dependabot alerts, secret scanning and
     Trivy scans of built images, with Renovate handling dependency update PRs.
   - Consider Trivy Operator later for deployed-workload coverage, with its
     configuration in this repo and a way to surface reports. Track host security
     separately; keep updates reviewed and preserve stateful backup requirements.
7. Set up branch protection for `main`: require all changes to go through pull
   requests and prevent direct pushes, including administrator bypasses.
8. Explore proxy-level SSO using Keycloak OIDC, with Envoy replacing Traefik.
9. Resolve the [known Proxmox NUT issue](nut-update.md) through a compatible
   official Debian fix; UPS communication loss and automatic power recovery
   also remain unverified. Existing shutdown protection stays enabled.
