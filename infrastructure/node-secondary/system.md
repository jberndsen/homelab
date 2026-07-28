# Current state

- Runs Ubuntu Server.
- Hostname: `nuc`.
- IP address: `192.168.1.33`.
- SSH user: `jeroen`.
- Architecture: `x86_64`.
- Kernel version observed on 2026-07-10: `6.8.0-107-generic`.
- Runs workloads with Docker Compose, fully defined in @docker-compose.yml and @.env.
- Docker Compose workloads use UID `1000` (`PUID`) and GID `988` (`PGID`).
- On this host, UID `1000` is user `jeroen` and GID `988` is group `docker`.
- Verified on 2026-07-10: containers running as UID `1000` and GID `988` have read-write access to the NFS-mounted downloads, movies, series, and music directories.

## Running workload image channels

Observed on 2026-07-10:

- `transmission`: `haugene/transmission-openvpn` (must be replaced with a non-VPN image in K3s).
- `homepage`: `latest`.
- `plex`, `radarr`, `sonarr`, and `lidarr`: LinuxServer images without an explicit tag.
- `prowlarr`: `lscr.io/linuxserver/prowlarr:develop`.

Prowlarr version observed on 2026-07-10: `2.3.1.5238` (`develop` pre-release). It will migrate directly to a pinned stable release that is newer than this build.

FlareSolverr was manually decommissioned on 2026-07-28. It is broken and will
not be migrated to K3s. Prowlarr restores must remove any retained FlareSolverr
proxy configuration.

Before the Transmission cutover, its active download queue will be manually drained. The migration will not copy Transmission runtime state or run the old and new download clients concurrently.
