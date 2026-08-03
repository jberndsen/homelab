# K3s media migration runbook

These files migrate the current `node-secondary` Compose stack one service at
a time. Run them in numeric order. Each Markdown step names the matching
multi-document manifest in [`manifests/`](manifests/) and contains its own
preconditions, owner confirmations, validation, and rollback procedure.

## Roles and stop rules

- **Agent:** may execute the command autonomously when asked to run that step.
- **Owner:** must perform or explicitly confirm the action. The agent must not
  infer the answer.
- **STOP:** do not continue to the next numbered step until the gate passes.
- Never delete a PVC or NAS data as rollback. Scale down the new Deployment and
  restart the intact old Compose service instead.
- Before stopping any stateful Compose service, create its application-native
  backup and download the backup to `plans/runtime/backups/`.
- `plans/runtime/` is operational, untracked material. It must contain secrets
  and backups, never committed manifests.

## Fixed design

- Namespace: `media`.
- NAS: one retained NFSv3 PV for
  `192.168.1.32:/var/nfs/shared/media`, mounted once as `/data`.
- Real paths: `/data/Downloads`, `/data/Video/Movies`, and
  `/data/Video/TV Shows`.
- App configuration: one local-path PVC per stateful application.
- LinuxServer images: `PUID=1000`, `PGID=988`, `UMASK=002`, and
  `TZ=Europe/Amsterdam`; do not force their init process to UID 1000.
- Exposure: ClusterIP Services and Traefik HTTP Ingress at `192.168.30.103`.
- DNS: individual UniFi A records under `home.arpa`.
- All K3s workload egress intentionally follows VLAN 30's NordVPN route for
  this first migration. A dedicated Transmission network identity is deferred.
- Existing image references are pinned to registry manifest digests resolved
  on 2026-07-10. Bazarr's stable image must be resolved to a reviewed
  immutable digest immediately before its implementation; later digest updates
  are separate reviewed operations.

Pinned image metadata at resolution time:

| Workload | Image version |
| --- | --- |
| Transmission (LinuxServer) | `4.1.3-r0-ls353` |
| Prowlarr | `2.4.0.5397-ls153` |
| Radarr | `6.2.1.10461-ls309` |
| Sonarr | `4.0.19.2979-ls319` |
| Bazarr | Resolve stable version and immutable digest at implementation time |
| Plex | `1.43.2.10687-563d026ea-ls312` |
| Homepage | `v1.13.2` |

## Execution order

1. [Preliminaries and storage](01-preliminaries-and-storage.md)
2. [Transmission](02-transmission.md)
3. [Prowlarr](03-prowlarr.md)
4. [Radarr](04-radarr.md)
5. [Sonarr](05-sonarr.md)
6. [Bazarr — new service](12-bazarr.md)
7. [Plex](07-plex.md)
8. [Homepage](08-homepage.md)
9. [Image updates and Watchtower decision](09-image-updates.md)
10. [Final acceptance and retirement](10-final-acceptance.md)

## Common commands

Run from the repository root:

```bash
kubectl get nodes -o wide
kubectl -n media get pods,pvc,services,ingresses
kubectl -n media get events --sort-by=.lastTimestamp
```

The manifests intentionally use singleton `Recreate` Deployments for
SQLite/config writers. TCP startup/readiness probes avoid treating restored
application authentication as a failed health check.
