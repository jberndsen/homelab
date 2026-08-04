# Step 10: final acceptance and old-stack retirement gate

No Kubernetes YAML is applied in this verification step.

## Cluster-wide acceptance

Agent:

```bash
kubectl get nodes -o wide
kubectl -n media get deployments,pods,services,ingresses,pvc
kubectl get pv media-nfs
kubectl -n media get events --sort-by=.lastTimestamp
```

Required:

- node Ready; every Deployment `1/1`; no restart loop or warning event
- every local config PVC Bound; `media-nfs` Bound with reclaim policy Retain
- each hostname works from Default and Servers VLAN clients
- Transmission uses NordVPN and fails closed
- Prowlarr indexers and both application integrations test successfully
- Radarr/Sonarr import and hardlink tests passed
- Jellyfin direct-play browser and native-client playback passed. The owner
  accepts a direct-play-only media policy; no transcoding capacity is required
  or assumed, and incompatible media must be manually re-downloaded.
- every Homepage link/widget works without runtime socket or Kubernetes RBAC
- VM filesystem has safe free space after Jellyfin metadata/cache creation

The legacy Plex Compose service was stopped on 2026-08-04 after owner
acceptance of Jellyfin. Keep its stopped container, Docker volumes, app
backups, and K3s PVCs intact as rollback until the owner explicitly chooses a
later retirement date. No deletion is authorized by this plan.

Update the relevant `infrastructure/*/system.md` after each executed step with
actual versions, observed public IP, NFS identity/modes, DNS records, backup
names, and acceptance results. Those records, not this pre-execution plan,
become the final system of record.
