# Step 2: Transmission

## Preconditions and owner gate

- Steps 1 passed.
- Owner confirms the old Transmission queue is empty. Check again immediately
  before cutover; do not run both clients against an active queue.
- Owner chooses a new RPC password; do not reuse the password committed in the
  old `.env`.

Create the untracked Secret file `plans/runtime/transmission-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: transmission-rpc
  namespace: media
type: Opaque
stringData:
  username: jeroen
  password: REPLACE_WITH_NEW_PASSWORD
```

Then:

```bash
chmod 600 plans/runtime/transmission-secret.yaml
kubectl apply -f plans/runtime/transmission-secret.yaml
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose stop transmission'
kubectl apply -f plans/manifests/02-transmission.yaml
kubectl -n media rollout status deployment/transmission --timeout=300s
```

The fresh instance starts with automatic torrent start and the download queue
disabled. It cannot begin normal downloading before the network gate.

## Configure and test

Open `http://transmission.home.arpa` and confirm RPC authentication. Preserve
the old non-VPN behavior using Preferences or RPC:

- completed: `/data/Downloads/completed`
- incomplete enabled: `/data/Downloads/incomplete`
- watch directory enabled: `/data/Downloads/watch`
- peer port `51413`; no port forwarding; TCP enabled; µTP disabled
- DHT and PEX enabled; local peer discovery disabled
- encryption preferred
- global peers `240`, per torrent `60`
- ratio limit `0.10`; idle seeding limit `2` minutes
- download queue size `15`; seed queue disabled
- no upload/download speed limits; umask `002`
- RPC authentication required; RPC host/IP whitelists disabled

Agent verifies the real Pod uses the VPN address:

```bash
kubectl -n media exec deployment/transmission -- curl -fsS https://api.ipify.org
```

Owner repeats the controlled UniFi VPN-unavailable test from Step 1. Public
egress from this Pod must fail without WAN fallback; NAS, DNS, UI, and cluster
access must remain available. Restore the VPN afterward.

Owner then enables automatic torrent start and the download queue. Add one
small legal test torrent, verify it completes under `/data/Downloads/completed`,
and confirm created files are writable as the shared identity.

Keep the stopped old container and volume intact throughout validation.

## Rollback

```bash
kubectl -n media scale deployment/transmission --replicas=0
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose start transmission'
```

Never restart the old instance until the new test queue is empty/removed. Do
not delete `transmission-config` or NAS content.
