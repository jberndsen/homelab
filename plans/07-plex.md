# Step 7: Plex

Plex is a fresh server. Do not copy its Compose volume, database, metadata, or
watch history. The old server stays running until new-client playback passes.

## Record and deploy

Owner records old library names/types, language/agent choices, and confirms the
folders are Movies and TV Shows. Obtain a fresh claim token immediately before
deployment; it expires within minutes. Create `plans/runtime/plex-claim.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: plex-claim
  namespace: media
type: Opaque
stringData:
  token: claim-REPLACE_IMMEDIATELY
```

Then:

```bash
chmod 600 plans/runtime/plex-claim.yaml
kubectl apply -f plans/runtime/plex-claim.yaml
kubectl apply -k infrastructure/node-main/08-plex
kubectl -n media rollout status deployment/plex --timeout=600s
kubectl -n media logs deployment/plex
```

## Configure and accept

At `http://plex.home.arpa`:

1. Confirm the server is claimed.
2. Keep Remote Access and DLNA disabled; do not publish/open TCP 32400.
3. Set the custom server access URL to `http://plex.home.arpa`.
4. Create Movies at `/data/Video/Movies` and TV at
   `/data/Video/TV Shows`, preserving recorded language/agent choices.
5. Run a full scan. The `/data` mount is read-only.
6. Test browser playback and at least one representative native LAN client.
   Confirm direct play and one software-transcode case if transcoding is used.

Remove the short-lived claim token from the running environment after claim:

```bash
kubectl -n media delete secret plex-claim
kubectl -n media rollout restart deployment/plex
kubectl -n media rollout status deployment/plex --timeout=600s
```

Re-test browser and native playback. Only then stop old Plex:

```bash
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose stop plex'
```

Monitor VM disk use; 30 GiB is a starting allocation and Plex metadata can
grow beyond it.

## Rollback

```bash
kubectl -n media scale deployment/plex --replicas=0
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose start plex'
```

Do not delete `plex-config`; keep it for diagnosis even though the old server
has independent state.
