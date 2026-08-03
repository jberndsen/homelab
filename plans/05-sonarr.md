# Step 5: Sonarr

## Backup and cutover

In old Sonarr, create **System → Backup → Backup Now**, download the ZIP
to `plans/runtime/backups/sonarr/`, then:

```bash
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose stop sonarr'
kubectl apply -f plans/manifests/05-sonarr.yaml
kubectl -n media rollout status deployment/sonarr --timeout=300s
```

## Restore and acceptance

At `http://sonarr.home.arpa` restore the ZIP, then:

1. Add `/data/Video/TV Shows` as a root folder. Then use **Series → Mass
   Editor** to select every existing series and change its root folder to that
   path. Do not enable any option that moves existing files: this is a database
   path reassignment because the files are already on the NAS.
2. Run a refresh/scan after the reassignment. In **System → Tasks**, run
   **Check Health** and confirm there is no missing `/series` root-folder
   warning before removing the old root-folder record.
3. Configure/test Transmission at `http://transmission:9091`; use the existing
   series category if applicable. Add no Remote Path Mapping.
4. Update Prowlarr's Sonarr URL to `http://sonarr:8989`, test, then enable the
   intended sync level.
5. Rescan and import one controlled completed download. Confirm expected
   source/destination inode equality, ownership/mode, and application health.

STOP on any failed gate.

## Rollback

```bash
kubectl -n media scale deployment/sonarr --replicas=0
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose start sonarr'
```

Disable the new Prowlarr integration first. Keep PVC and backup.
