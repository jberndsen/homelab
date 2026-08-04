# Current state

- `node-secondary` is the Ubuntu Server host `nuc` at `192.168.1.33`.
- SSH user: `jeroen`.
- Docker Compose remains installed. The legacy Homepage service was stopped
  on 2026-08-04 after its K3s replacement passed owner browser/widget
  acceptance. Homepage and Plex are stopped and retained intact as rollback
  paths; no legacy Docker Compose services are running.
- The legacy Plex Compose service was stopped on 2026-08-04 after the owner
  accepted Jellyfin as the direct-play-only K3s media server. Preserve Plex's
  stopped container, Compose configuration, and data as the rollback path;
  do not remove them without explicit owner approval.
- Docker Compose workloads use UID `1000` (`jeroen`) and GID `988` (`docker`).
