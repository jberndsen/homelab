# Step 8: Homepage

Homepage is recreated from source-controlled ConfigMap data. It receives no
Docker/containerd socket and no Kubernetes API RBAC. Its five app widgets are
preserved.

## Secret and deploy

Collect the current API key from each restored Servarr app and a Plex token.
Create `plans/runtime/homepage-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: homepage-widgets
  namespace: media
type: Opaque
stringData:
  HOMEPAGE_VAR_PLEX_KEY: REPLACE
  HOMEPAGE_VAR_SONARR_KEY: REPLACE
  HOMEPAGE_VAR_RADARR_KEY: REPLACE
  HOMEPAGE_VAR_LIDARR_KEY: REPLACE
  HOMEPAGE_VAR_PROWLARR_KEY: REPLACE
```

Apply:

```bash
chmod 600 plans/runtime/homepage-secret.yaml
kubectl apply -f plans/runtime/homepage-secret.yaml
kubectl apply -f plans/manifests/08-homepage.yaml
kubectl -n media rollout status deployment/homepage --timeout=300s
```

## Acceptance and cutover

At `http://homepage.home.arpa`, test every link and all five widgets. Confirm
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
