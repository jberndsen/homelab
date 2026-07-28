# Current state

- `node-secondary` is the Ubuntu Server host `nuc` at `192.168.1.33`.
- SSH user: `jeroen`.
- Docker Compose remains installed for the two running legacy services:
  Plex and Homepage.
- Plex is retained as the legacy media server while Jellyfin is the active K3s
  media server. Do not stop or remove Plex unless the owner explicitly asks.
- Docker Compose workloads use UID `1000` (`jeroen`) and GID `988` (`docker`).
