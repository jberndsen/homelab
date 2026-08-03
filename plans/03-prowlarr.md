# Step 3: Prowlarr

## Backup and cutover

Owner, in old Prowlarr (`http://192.168.1.33:9696`):

1. **System → Backup → Backup Now**.
2. Download the ZIP to `plans/runtime/backups/prowlarr/` and record its name.
3. Stop old Prowlarr:

```bash
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose stop prowlarr'
# Apply and validate one resource at a time. The local-path PVC may remain
# Pending until the Deployment creates its first consumer.
kubectl apply -f infrastructure/node-main/05-prowlarr/01-config-persistentvolumeclaim.yaml
kubectl -n media get pvc prowlarr-config
kubectl apply -f infrastructure/node-main/05-prowlarr/02-deployment.yaml
kubectl -n media rollout status deployment/prowlarr --timeout=300s
kubectl -n media get pvc prowlarr-config
kubectl apply -f infrastructure/node-main/05-prowlarr/03-service.yaml
kubectl -n media get endpointslice -l kubernetes.io/service-name=prowlarr
kubectl apply -f infrastructure/node-main/05-prowlarr/04-ingress.yaml
curl -fsS -o /dev/null http://prowlarr.home.arpa
```

The target is stable Prowlarr `2.4.0.5397`, newer than the old develop build
`2.3.1.5238`.

## Restore and acceptance

At `http://prowlarr.home.arpa`:

1. **System → Backup → Restore Backup**, select the downloaded ZIP.
2. Wait for restart; verify version, authentication, logs, and health.
3. Disable or leave failing old Radarr/Sonarr application integrations
   until each destination step exists. Do not run Full Sync to stale URLs.
4. Remove any restored FlareSolverr proxy configuration. FlareSolverr was
   intentionally decommissioned and will not be migrated.
5. Test every relevant indexer.

STOP on database downgrade errors, failed indexers, or missing restored data.

## Rollback

```bash
kubectl -n media scale deployment/prowlarr --replicas=0
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose start prowlarr'
```

Keep `prowlarr-config` and the backup ZIP.
