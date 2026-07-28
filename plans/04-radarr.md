# Step 4: Radarr

## Backup and cutover

In old Radarr, create **System → Backup → Backup Now**, download the ZIP
to `plans/runtime/backups/radarr/`, then:

```bash
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose stop radarr'
kubectl apply -f plans/manifests/04-radarr.yaml
kubectl -n media rollout status deployment/radarr --timeout=300s
```

## Restore and acceptance

At `http://radarr.home.arpa` restore the ZIP through **System → Backup →
Restore Backup**, then:

1. Change the root folder to `/data/Video/Movies` and move all movie entries to
   that root without moving files on disk.
2. Configure/test Transmission at `http://transmission:9091` with the new RPC
   credentials and a movie category if one was previously used.
3. Do not add a Remote Path Mapping: both apps see `/data/Downloads/...`.
4. In Prowlarr, add/update Radarr to `http://radarr:7878`, test it, then enable
   the intended sync level.
5. Rescan, import one controlled completed download, and confirm the source and
   movie destination have the same inode (`stat -c %i`) when hardlinking is
   expected. Verify owner/group/mode and health.

STOP before proceeding if restore, root paths, import, or hardlink validation
fails.

## Rollback

```bash
kubectl -n media scale deployment/radarr --replicas=0
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose start radarr'
```

Disable the new Prowlarr integration before restarting old Radarr. Preserve the
PVC and backup.
