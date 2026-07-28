# Step 3: Prowlarr

## Backup and cutover

Owner, in old Prowlarr (`http://192.168.1.33:9696`):

1. **System → Backup → Backup Now**.
2. Download the ZIP to `plans/runtime/backups/prowlarr/` and record its name.
3. Stop old Prowlarr:

```bash
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose stop prowlarr'
kubectl apply -f plans/manifests/03-prowlarr.yaml
kubectl -n media rollout status deployment/prowlarr --timeout=300s
```

The target is stable Prowlarr `2.4.0.5397`, newer than the old develop build
`2.3.1.5238`.

## Restore and acceptance

At `http://prowlarr.home.arpa`:

1. **System → Backup → Restore Backup**, select the downloaded ZIP.
2. Wait for restart; verify version, authentication, logs, and health.
3. Disable or leave failing old Radarr/Sonarr/Lidarr application integrations
   until each destination step exists. Do not run Full Sync to stale URLs.
4. Test every relevant indexer.

STOP on database downgrade errors, failed indexers, or missing restored data.

## Rollback

```bash
kubectl -n media scale deployment/prowlarr --replicas=0
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose start prowlarr'
```

Keep `prowlarr-config` and the backup ZIP.
