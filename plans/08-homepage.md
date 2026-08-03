# Step 8: Homepage

Homepage is recreated from source-controlled ConfigMap data. It receives no
Docker/containerd socket and no Kubernetes API RBAC. Its four app widgets are
preserved.

## Migration status — 2026-07-28

- The K3s manifests are deployed from `infrastructure/node-main/08-homepage/`.
  They include an ignored local `homepage-secret.yaml`, which is included by
  the Kustomization and contains the four widget API keys.
- The deployment, ClusterIP Service, and Traefik Ingress are healthy. The
  page at `http://homepage.home.arpa` returned HTTP 200 three consecutive
  times through the ingress after deployment.
- A page-rendering 500 was corrected by supplying Homepage's required empty
  Docker/Kubernetes discovery config files and a writable ephemeral
  `/app/config/logs` directory. No runtime socket, Kubernetes API RBAC, or
  discovery configuration was added.
- Browser acceptance is still pending. The owner reported a `no available
  server` message in the Homepage UI after the 500 was fixed. Diagnose that
  message and verify the Jellyfin, Sonarr, Radarr, and Prowlarr links/widgets
  before cutover.
- The legacy Homepage container on `node-secondary` remains running as the
  rollback path. Do not stop it until the browser acceptance checks pass.

## Secret and deploy

Collect the current API key from Jellyfin and each restored Servarr app.
Populate the ignored local
`infrastructure/node-main/08-homepage/homepage-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: homepage-widgets
  namespace: media
type: Opaque
stringData:
  HOMEPAGE_VAR_JELLYFIN_KEY: REPLACE
  HOMEPAGE_VAR_SONARR_KEY: REPLACE
  HOMEPAGE_VAR_RADARR_KEY: REPLACE
  HOMEPAGE_VAR_PROWLARR_KEY: REPLACE
```

Apply:

```bash
chmod 600 infrastructure/node-main/08-homepage/homepage-secret.yaml
kubectl apply -k infrastructure/node-main/08-homepage
kubectl -n media rollout status deployment/homepage --timeout=300s
```

## Acceptance and cutover

At `http://homepage.home.arpa`, test every link and all four widgets. Confirm
there is no Docker/Kubernetes discovery/status expectation. Check health:

```bash
kubectl -n media get pod,service,ingress -l app.kubernetes.io/name=homepage
kubectl -n media logs deployment/homepage
```

Only after acceptance:

```bash
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose stop homepage'
```

## Rollback

```bash
kubectl -n media scale deployment/homepage --replicas=0
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose start homepage'
```
