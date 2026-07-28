# Jellyfin experiment

This is an evaluation of Jellyfin as a possible Plex replacement, not a
numbered migration cutover. It must run alongside the new K3s Plex deployment
and the still-running old `nuc` Plex server. Do not stop, scale down, alter,
or copy configuration from either Plex server while this experiment is active.

## Objective and acceptance question

Determine whether Jellyfin can play the existing NAS Movies and TV libraries
from the laptop on `192.168.1.118/24` and representative Plex-client subnets
through the existing Traefik ingress, without Plex's Remote Watch Pass or Plex
Pass requirement. This is a user-experience test, not a claim that Jellyfin
will satisfy every Plex feature or client requirement.

Jellyfin has no account-claim-token flow. Create a fresh local administrator
through its first-run setup; do not import Plex metadata, user data, watch
history, or configuration.

## Preconditions and owner gates

1. **Owner:** confirm that this parallel experiment is acceptable and that the
   NAS libraries remain read-only to Jellyfin.
2. **Agent:** verify that the K3s node remains Ready, `media-nfs` is Bound,
   Traefik remains reachable at `192.168.30.103`, and current Plex workloads
   are healthy. Stop if the existing migration has an unrelated failure.
3. **Agent:** inspect free VM storage before choosing the Jellyfin local-state
   and transcode-cache requests. `local-path` PVC requests are not hard quotas;
   do not assume the original Plex storage observation is still available.
4. **Owner:** create the UniFi DNS record `jellyfin.home.arpa` pointing to
   `192.168.30.103`, and confirm resolution from the laptop and one other
   intended client subnet.
5. **Agent:** resolve a current stable `jellyfin/jellyfin` image to an immutable
   manifest digest, record the resolved version/digest in the review, and use
   that digest in the experiment manifests. Do not deploy a floating tag.

## Proposed isolated Kubernetes design

Create a separate `infrastructure/node-main/09-jellyfin-experiment/`
Kustomization only after the above gates pass:

- Namespace: `media`; labels: `app.kubernetes.io/name: jellyfin` and
  `app.kubernetes.io/part-of: media-stack`.
- One singleton `Recreate` Deployment using the official `jellyfin/jellyfin`
  image pinned by digest, with `automountServiceAccountToken: false`.
- Dedicated node-local PVC(s) for `/config` and, if a persistent cache is
  chosen after the capacity check, `/cache`; never store either on NFS. Keep
  the final requested sizes explicit in the manifests and system record.
- Mount `media-nfs` once at `/data`, read-only. Create a Movies library at
  `/data/Video/Movies` and a TV library at `/data/Video/TV Shows`.
- A ClusterIP Service on Jellyfin's HTTP port `8096` and a Traefik Ingress for
  `jellyfin.home.arpa`. Do not publish direct application ports, use host
  networking, enable DLNA, or create an external/NAT port-forward.
- Use software transcoding only. The VM has no `/dev/dri/renderD*` node, so do
  not pass GPU devices into the Pod or configure hardware acceleration.
- Do not grant Kubernetes RBAC, mount runtime sockets, or give Jellyfin
  write access to NAS media.

The official Jellyfin container supports separate persistent `/config` and
`/cache` storage and read-only media mounts. Its official reverse-proxy
guidance supports a subdomain route, which matches the existing Traefik design.

## Deploy and configure

After manifests are reviewed and the DNS gate passes:

```bash
kubectl apply -k infrastructure/node-main/09-jellyfin-experiment
kubectl -n media rollout status deployment/jellyfin --timeout=600s
kubectl -n media get pod,service,ingress,pvc -l app.kubernetes.io/name=jellyfin
kubectl -n media logs deployment/jellyfin
```

At `http://jellyfin.home.arpa`:

1. Create the first local administrator and store its credentials only in the
   approved password manager, not this repository.
2. Create fresh Movies and TV libraries using the two `/data/Video/...` paths.
3. Allow the initial scan to finish. Verify that file browsing and metadata
   matching do not require NAS write access.
4. Keep DLNA disabled. Do not configure internet exposure, NAT forwarding, or
   a public DNS name for this experiment.

## Acceptance tests

Run each check before considering Jellyfin as a replacement candidate:

1. From `192.168.1.118`, open `http://jellyfin.home.arpa`, authenticate, and
   play one movie and one TV episode without a paid Plex subscription prompt.
2. Repeat browser or native-client playback from every other intended client
   subnet. Record the client subnet and result; do not infer that one VLAN
   proves the others.
3. Confirm direct play of a representative file and one intentionally chosen
   software-transcode case. Monitor Pod logs and VM CPU/RAM during the latter.
4. Confirm that no direct `8096` LAN port, DLNA listener, or remote/NAT
   exposure was introduced.
5. Compare the required client experience with Plex: library browsing,
   playback, account/user needs, and any features that matter to the owner.

## Decision, rollback, and cleanup

The owner makes the replacement decision only after all acceptance tests pass.
If Jellyfin is accepted, write a dedicated Plex retirement/cutover plan before
stopping either Plex server or changing Homepage widgets. If it is rejected:

```bash
kubectl -n media scale deployment/jellyfin --replicas=0
```

Keep the Jellyfin local PVCs for diagnosis until the owner explicitly approves
their deletion. Never delete NAS media. Restarting or retaining either Plex
server remains independent of this experiment.
