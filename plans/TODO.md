# Next steps

1. Add a finance namespace with [Budgero](https://budgero.app/docs/self-hosting-guide)
   to try replacing YNAB.
2. Add an `immich` namespace with [Immich](https://docs.immich.app/install/kubernetes/)
   for an additional copy of iPhone photos, retaining existing originals.
   Follow the [Immich implementation handoff](immich.md) one step per agent
   session; [execution progress](immich-progress.md) starts at step 1. Scope
   includes media/Immich resource allocation and Grafana headroom visibility.
3. Plan how to reinstall the second node with Proxmox and use it for cluster
   capacity or backup.
4. Complete [Home Assistant validation and follow-ups](home-assistant-stability.md):
   post-update smoke checks, P1 diagnostics, Hue IoT migration and recovery.
   Preserve the existing installation; resume metrics through the separate
   [owner-led Prometheus teaching plan](HA-prometheus-connection.md).
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
8. Investigate [UPS communication loss](nut-incidents.md): correlate host,
   UPS-service and Study-switch events; if historical evidence is unavailable,
   define bounded, redacted monitoring for the next event. The stale-CAL fix is installed.
9. **Solve automatic physical restart after power restoration.** Confirm the
   NUC's BIOS AC-recovery/WOL capabilities and UPS restore behavior; design a
   recovery path that also handles a gracefully halted host whose AC never
   disappeared. Account for the NAS and switch before considering UPS output
   cycling. Document the chosen setup and validate host power-on, VM startup and
   service recovery in an owner-approved maintenance window with verified backups
   and a manual recovery method. Guest onboot alone does not complete this task.
10. Retire the scoped NUT proposed-updates source once the fixed package reaches
    regular Debian sources, following [NUT maintenance](../infrastructure/node-main/nut/README.md#routine-updates-and-source-retirement).
11. Plan and implement offsite protection against NAS data loss using the
    owner's requested newly offered UniFi service with Backblaze. Confirm the
    service and its capabilities during that work. Protect critical data;
    exclude replaceable media files, with the exact selection confirmed before
    implementation. Include photo originals in the scope discussion, retention
    against accidental deletion, and a restore test; mirroring alone must not
    be assumed to provide recoverable history. This is separate from and out of
    scope for the Immich planning session.
    Revisit whether local NAS snapshots are worthwhile and choose retention
    with storage costs in mind; snapshots are deferred from the initial Immich
    deployment by owner decision.
12. Introduce trusted HTTPS consistently across internal applications,
    including Immich. Plan certificate issuance, client trust, renewal and
    ingress/DNS changes while preserving private access. Keep this separate
    from the initial Immich deployment, which retains internal HTTP, and
    coordinate with any future ingress/SSO changes in item 7.

## Optional monitoring and recovery follow-ups

The initial monitoring setup is complete; see [monitoring operations](../infrastructure/node-main/ns-monitoring/README.md).

- Observe retention and disk growth over a full week and measure peak usage.
- Plan a backup restore test, including application consistency and independent
  recovery sources; nightly full-VM backups currently remain untested.
- Add Proxmox, NAS or UPS metrics after confirming device interfaces and access.
- Design notifications, including an external observer for host/VM outages.
- Decide separately whether to retain Homepage. OpenTelemetry is outside the
  completed monitoring scope.
