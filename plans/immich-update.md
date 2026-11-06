# Immich update instructions for a clean agent

Use this document when the owner requests an Immich update. Read it together
with AGENTS.md and the deployed application's operations documentation.

Status: prepared during initial Immich planning; no Immich deployment exists
yet from this work. The initial implementation must validate and complete the
deployment-specific references below before marking this runbook ready. Do not
use this document to install Immich or pretend an update has been tested.
The implementation plan's step 8 owns finalizing this runbook. Its separate
one-step-per-session contract applies to initial implementation, not a routine
update requested through this document.

## Scope and prerequisites

- Read [the implementation decisions](immich.md), relevant
  infrastructure/**/system.md records, and the application's README under
  infrastructure/node-main/ns-immich once installed.
- Read [Argo operations](../infrastructure/node-main/ns-argo/SETUP.md) and
  [monitoring operations](../infrastructure/node-main/ns-monitoring/README.md).
- Use local kubectl with its configured context, starting with
  `kubectl get nodes -o wide`. Inspect Git status, Argo state, deployed image
  digests, workload health and current database backup status. Preserve unrelated
  changes. Do not print credentials, tokens or personal photo metadata.
- Keep all temporary files and downloaded references in repository-root tmp/;
  set TMPDIR to that absolute path. Never commit them.
- If the owner names a release, assess that release. Otherwise select the
  current stable release after reviewing supported upgrade paths and mobile
  compatibility. No prerelease or mutable latest/release tags.

## Review the complete release change

1. Identify the deployed server/ML release and exact database/extension and
   Valkey versions. Compare live state with Git; resolve unexplained drift first.
2. Review upstream release notes for every intervening version, upgrade guidance
   and breaking changes. iPhone apps update automatically; check their versions
   with the owner if not otherwise observable, including the wife's after she
   is onboarded. Use the selected release's compatibility policy, not an assumed
   universal exact-minor-version rule. Explain any client action needed.
3. Obtain the target release's Compose file and compare with the previous
   recorded reference. Compare a fixed official Helm chart revision against
   the recorded chart reference as well. Charts and application releases may
   advance independently. Never assume mutable main matches a release.
4. Check images, environment variables, configuration schema/defaults, paths,
   permissions, probes, ports, resources, database extensions, migrations,
   backup/restore procedures and metrics. Review applicable library-chart
   behavior if a chart change affects generated workloads. Do not install Helm.
5. Record the reviewed release/chart references and relevant differences in the
   update description. A Renovate image-only PR is not proof this review ran.

## Prepare the change and recovery point

- Update immich-server and immich-machine-learning as one reviewed release
  change, with matching release versions and exact pinned digests. Keep
  automerge off. This exception must be documented in AGENTS.md, Renovate and
  the update guides before use; the owner approved it in the implementation plan.
- Keep PostgreSQL and Valkey updates separate. If a target release requires a
  prerequisite database/extension migration, document and execute the supported
  sequence separately before the application update. Never substitute a new
  PostgreSQL major-version tag for a migration procedure.
- Translate upstream configuration changes into the existing Kustomize
  manifests. Preserve media-playback priority, NAS paths, user storage layout,
  private HTTP access, code-managed settings and agreed backup retention.
- Render affected overlays, validate application/Prometheus configuration as
  appropriate, and review server dry-run/diff. Check scope and secret hygiene.
  Review any Argo prune/delete action explicitly; do not delete PVCs or use
  force/replace as an update shortcut.
- Before pushing or merging a deploying change, take a fresh application-native
  database backup using the deployed release's supported procedure. Verify
  successful completion and the nonempty NAS file; record its identity privately
  and protect the pre-update recovery point from routine retention until the
  update is accepted. Confirm required asset-file availability and describe the
  limits of recovery protection. A VM backup does not replace this step.
- For changes that require a coordinated asset/database recovery point, follow
  upstream guidance and the application's documented procedure. Do not claim
  current NAS files plus a dump are an independent historical asset backup.
  If the supported upgrade needs protection unavailable under the owner's
  no-snapshots/no-offsite initial scope, explain that concrete gap and obtain
  a decision before the deploying change. Do not silently introduce snapshots.

## Deploy and verify

1. Follow the repository's current branch/PR workflow. Review the concrete diff
   and complete release/backup checks before pushing or merging to a branch
   Argo deploys. Do not manually apply Argo-owned workload changes.
2. Wait for Argo sync and each affected rollout. Inspect readiness, startup and
   migration errors, events and restart/OOM evidence. Verify actual running
   digests. Bound waits and investigate failures instead of retrying blindly.
3. Check browser and iPhone login, sample photo/video upload and original
   download, dates/locations, expected storage layout, ML/search and database
   backup operation. Use disposable assets for deletion checks and never alter
   the owner's real library to test an update. Verify existing media playback
   and relevant Grafana views/scrape targets; inspect resource contention.
4. Distinguish completed checks from owner-dependent iPhone checks still pending.
   Pod readiness alone is not a successful application update. Stop further
   rollout changes if material checks fail and investigate.

## Failure handling and documentation

Do not blindly roll back images: Immich does not support downgrades, and a
migrated database may not work with the previous release. Preserve evidence
and the pre-update recovery point. Use the release-compatible documented
recovery procedure, coordinating Argo automation before restoring. Destructive
restoration of production data requires a concrete owner-approved recovery
action; an update request does not authorize discarding library changes.

Update application setup/update/recovery documentation, upstream reference
versions and affected monitoring instructions. Update system records only for
durable changed facts, not discoverable image versions or usage snapshots.
Check links and commands. Report old/new versions, relevant upstream changes,
backup verification, rollout/smoke evidence and remaining limitations. Do not
claim recovery was tested unless it actually was; an isolated restore exercise
is not part of routine updates by default.

## Initial implementation must complete

- Link the actual application README, manifest/config locations and exact
  workload/service names after installation.
- Provide tested native-backup creation/status commands, backup location,
  recovery-point preservation steps and smoke-test commands in that README;
  link them here without duplicating conflicting instructions.
- Record fixed upstream application/Compose/chart references for future diffs.
- Verify the scoped Renovate grouping and repository-policy exception.
- Mark this runbook ready only after reviewing it against the installed setup.

Upstream references: [upgrading](https://docs.immich.app/install/upgrading/),
[backup and restore](https://docs.immich.app/administration/backup-and-restore/),
[Compose installation](https://docs.immich.app/install/docker-compose/),
[official chart](https://github.com/immich-app/immich-charts).
