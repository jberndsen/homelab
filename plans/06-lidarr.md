# Step 6: Lidarr

## Backup and cutover

In old Lidarr, create **System → Backup → Backup Now**, download the ZIP
to `plans/runtime/backups/lidarr/`, then:

```bash
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose stop lidarr'
kubectl apply -f plans/manifests/06-lidarr.yaml
kubectl -n media rollout status deployment/lidarr --timeout=300s
```

## Restore and acceptance

At `http://lidarr.home.arpa` restore the ZIP, then:

1. Change the music root to `/data/Music/Lossless` and update artist entries
   without moving existing files.
2. Configure/test Transmission at `http://transmission:9091`; use the existing
   music category if applicable. Add no Remote Path Mapping.
3. Update Prowlarr's Lidarr URL to `http://lidarr:8686`, test, then enable the
   intended sync level.
4. Rescan and import one controlled completed download. Confirm expected
   source/destination inode equality, ownership/mode, and application health.

STOP on any failed gate.

## Rollback

```bash
kubectl -n media scale deployment/lidarr --replicas=0
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose start lidarr'
```

Disable the new Prowlarr integration first. Keep PVC and backup.
