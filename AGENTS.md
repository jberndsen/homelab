# Goal

Migrate workloads from Docker Compose on `node-secondary` to K3s in an Ubuntu VM on `node-main`. Always consult the documentation in infrastucture folders to determine the current system layout. Whenever unsure about node system information, workload details or network information, do not guess but ask me. I will look it up and we'll add it to the system documentation as system of record. The goal is to end up with structured Kubernetes YAML config files in this repository, under `./infrastructure/node-main`.

The current system-of-record files are the `system.md` documents under
`infrastructure/`. Historical migration plans and research are not current
system state.

# Migration approach

- Migrate Docker Compose workloads from `node-secondary` to K3s one application at a time.
- Deploy each application fresh on K3s; do not copy Docker volumes or container-level configuration directories.
- For applications whose configuration must be retained, use the application's manual backup/export and restore workflow. Provide step-by-step guidance when migrating each application.
- Keep media on the NAS and mount the existing directories into K3s workloads over NFS.
- Use a Transmission image without an integrated VPN client. VLAN ID `30` routes its traffic through the configured UniFi NordVPN NordLynx client.

## Dependency-ordered migration sequence

1. **NFS storage foundation** — mount or otherwise expose the NAS `media` share to K3s as persistent storage before migrating any workload that reads or writes media. Prefer a PersistentVolume declaration instead of mounting on the host machine.
2. **Transmission** — deploy a non-VPN image and configure its NAS downloads directory. Radarr and Sonarr require an available download client.
3. **Prowlarr** — restore its configuration after Transmission is available. It supplies indexer configuration to Radarr and Sonarr.
4. **Radarr** — restore its configuration after Prowlarr and Transmission are available; mount the NAS downloads and movies directories.
5. **Sonarr** — restore its configuration after Prowlarr and Transmission are available; mount the NAS downloads and series directories.
6. **Jellyfin** — deploy with the NAS movies and series directories mounted read-only. It is the active K3s media server; direct play has been validated from the laptop and NVIDIA Shield client, while software transcoding remains unvalidated.
7. **Homepage** — deploy last, once the service endpoints are stable, then recreate dashboard links and widgets.
8. **Watchtower replacement decision** — do not migrate the Docker-socket-based Watchtower container to K3s. Decide separately how Kubernetes workload image updates will be managed.


## Cluster access

Use the locally installed `kubectl` and its preconfigured kubeconfig to work with the K3s cluster. Start cluster work by checking node readiness with `kubectl get nodes -o wide`.
