# Step 1: preliminaries, network, Secrets encryption, and NFS

This step is complete only when every gate below passes. It makes no
application cutover.

## 1. Cluster and VM baseline

Agent:

```bash
kubectl get nodes -o wide
kubectl -n kube-system get deploy,daemonset,service,pods -o wide
kubectl get storageclass,ingressclass
```

Required observations:

- `vm-ubuntu-k3s` is `Ready`.
- Traefik advertises `192.168.30.103` on ports 80/443.
- `local-path` is the default StorageClass.

Owner, on `192.168.30.103`, install the currently missing NFS client package:

```bash
sudo apt-get update
sudo apt-get install --yes nfs-common
command -v mount.nfs
showmount -e 192.168.1.32
```

STOP if the parent media export is not available to `192.168.30.103`.

## 2. Enable K3s Secrets encryption

Encryption is currently disabled. These commands follow the K3s procedure for
enabling encryption on an existing single-server cluster in the
[official K3s procedure](https://docs.k3s.io/cli/secrets-encrypt#enable-secrets-encryption-on-an-existing-cluster).
Owner performs the sudo steps and must preserve any existing K3s config.

```bash
sudo k3s secrets-encrypt status
sudo k3s secrets-encrypt enable
sudo install -d -m 0755 /etc/rancher/k3s
sudo touch /etc/rancher/k3s/config.yaml
sudoedit /etc/rancher/k3s/config.yaml
```

Add this key without deleting or duplicating other configuration:

```yaml
secrets-encryption: true
```

Then:

```bash
sudo systemctl restart k3s
sudo k3s secrets-encrypt status
sudo k3s secrets-encrypt rotate-keys
sudo systemctl restart k3s
sudo k3s secrets-encrypt status
kubectl get nodes -o wide
```

Required final status:

- `Encryption Status: Enabled`
- `Current Rotation Stage: reencrypt_finished`
- `Server Encryption Hashes: All hashes match`
- node returns to `Ready`

STOP if any condition is absent. Do not create application Secrets yet.

## 3. UniFi VPN routing dependency

The recorded UniFi setup is Cloud Gateway Max, OS `5.1.19`, Network `10.4.57`.
The policy source is `192.168.30.0/24`, the NordLynx client is selected, and
Kill Switch is enabled. All K3s workloads intentionally use this route.

Owner decision on 2026-07-28: fail-closed NordLynx behavior is managed by
UniFi Network and is an accepted external network-service dependency, not a
K3s migration validation gate. Its configuration and this scope decision are
recorded in `infrastructure/network/system.md`.

## 4. Validate the NFS parent mount and hardlinks

The manifest creates one retained PV/PVC and an `nfs-behavior-test` Job. The
Job runs as UID:GID `1000:988`, exercises create/write/sync/rename/read/delete,
and creates cross-directory hardlinks into all three library roots. It cleans
up its uniquely named files on exit.

```bash
kubectl -n media wait --for=condition=complete job/nfs-behavior-test --timeout=180s
kubectl -n media logs job/nfs-behavior-test
kubectl get pv media-nfs
kubectl -n media get pvc media-nfs
```

Required:

- Job log ends in `NFS_BEHAVIOR_TEST_PASSED`.
- Every hardlink reports the same inode as its source.
- `media-nfs` PV/PVC is `Bound` and the PV reclaim policy is `Retain`.

The share mode is unknown; this behavioral result is authoritative for the
migration. Do not change the UNAS share mode to make a failed test pass. STOP
and diagnose permissions/export behavior instead.

After recording logs:

```bash
kubectl -n media delete pod vpn-route-test
kubectl -n media delete job nfs-behavior-test
```

Do not delete the PV or PVC.

Prepare the untracked backup directories used by later steps:

```bash
mkdir -p plans/runtime/backups/{prowlarr,radarr,sonarr}
```

## 5. Create LAN DNS record

Owner creates the UniFi Gateway wildcard Host (A) record `*.home.arpa`,
pointing to `192.168.30.103`. It was verified on 2026-07-28 by opening
`http://ingress-test.home.arpa`, which reached the temporary Traefik ingress
test successfully. Each future application name resolves through this wildcard;
an HTTP 404 is expected until its matching Ingress exists.

## Rollback

- Secrets-encryption rollback is not part of this migration. Stop and diagnose
  before creating application Secrets if enablement fails.
- Delete only the disposable Pod/Job after preserving logs.
- Never delete `media-nfs` or change the NAS share mode during diagnosis.
