# Next steps

1. Add cluster and workload observability. Decide separately whether to keep Homepage. Prometheus, Grafana, OpenTelemetry.
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
8. SSO using Keycloak OIDC by setting it up on the proxies instead of the apps itself, use Envoy instead of Traefik.