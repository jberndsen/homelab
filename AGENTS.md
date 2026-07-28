# Goal

Migrate workloads from Docker Compose on `node-secondary` to K3s in an Ubuntu VM on `node-main`. Always consult the documentation in infrastucture folders to determine the current system layout. Whenever unsure about node system information, workload details or network information, do not guess but ask me. I will look it up and we'll add it to the system documentation as system of record. The goal is to end up with structured Kubernetes YAML config files in this repository, under `./infrastructure/node-main`.

The agreed migration design and per-application cutover procedure are in [migration-plan.md](migration-plan.md). The primary-source evidence behind that plan is in [infrastructure/research/k3s-media-migration.md](infrastructure/research/k3s-media-migration.md).

# Migration approach

- Migrate Docker Compose workloads from `node-secondary` to K3s one application at a time.
- Deploy each application fresh on K3s; do not copy Docker volumes or container-level configuration directories.
- For applications whose configuration must be retained, use the application's manual backup/export and restore workflow. Provide step-by-step guidance when migrating each application.
- Watch history does not need to be retained.
- Keep media on the NAS and mount the existing directories into K3s workloads over NFS.
- Use a Transmission image without an integrated VPN client. VLAN ID `30` routes its traffic through the configured UniFi NordVPN NordLynx client.

## Dependency-ordered migration sequence

1. **NFS storage foundation** — mount or otherwise expose the NAS `media` share to K3s as persistent storage before migrating any workload that reads or writes media. Prefer a PersistentVolume declaration instead of mounting on the host machine.
2. **Transmission** — deploy a non-VPN image and configure its NAS downloads directory. Radarr, Sonarr, and Lidarr require an available download client.
3. **Prowlarr** — restore its configuration after Transmission is available. It supplies indexer configuration to Radarr, Sonarr, and Lidarr.
4. **Radarr** — restore its configuration after Prowlarr and Transmission are available; mount the NAS downloads and movies directories.
5. **Sonarr** — restore its configuration after Prowlarr and Transmission are available; mount the NAS downloads and series directories.
6. **Lidarr** — restore its configuration after Prowlarr and Transmission are available; mount the NAS downloads and music directories.
7. **Plex** — deploy with the NAS movies and series directories mounted; recreate its libraries and allow a fresh media scan. Watch history is intentionally not retained.
8. **Homepage** — deploy last, once the service endpoints are stable, then recreate dashboard links and widgets.
9. **Watchtower replacement decision** — do not migrate the Docker-socket-based Watchtower container to K3s. Decide separately how Kubernetes workload image updates will be managed.


## Cluster access

Use the locally installed `kubectl` and its preconfigured kubeconfig to work with the K3s cluster. Start cluster work by checking node readiness with `kubectl get nodes -o wide`.
