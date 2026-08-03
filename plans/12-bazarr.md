# Bazarr — new K3s service

Bazarr is a new service, not a migration. Do **not** create or restore a
backup, stop a Compose container, or copy a configuration directory. It
manages subtitles only for the films and episodes already indexed by the
accepted K3s Radarr and Sonarr instances.

## Agreed configuration

- Deploy the LinuxServer Bazarr image as a single `Recreate` Deployment,
  pinned to a reviewed immutable digest at implementation time.
- Keep `/config` on a new `5Gi` `local-path` PVC named `bazarr-config`.
- Mount `media-nfs` once, read/write, at `/data`; Bazarr writes sidecar
  subtitle files alongside the NAS media. Retain the common LinuxServer
  identity: `PUID=1000`, `PGID=988`, `UMASK=002`, and
  `TZ=Europe/Amsterdam`.
- Expose only a ClusterIP Service on port `6767` and a Traefik HTTP Ingress
  for `bazarr.home.arpa`; do not expose a direct LAN port.
- Create a fresh Bazarr administrator and retain the credentials only in the
  password manager.
- Configure a single English-only default language profile for all existing
  and future Radarr movies and Sonarr episodes. Embedded English tracks
  satisfy the requirement, so external subtitles are downloaded only when
  needed.
- Use OpenSubtitles.com as the sole initial provider. Its credentials stay in
  Bazarr's local configuration, not in version control. Do not configure
  OpenSubtitles.org: Bazarr supports it only for VIP users.
- Connect through in-cluster DNS using the existing Radarr and Sonarr API
  keys: `http://radarr:7878` and `http://sonarr:8989`.

Bazarr does not independently discover media; it works from the media that
Radarr and Sonarr index. Its upstream project documents both the Radarr/Sonarr
relationship and its ability to inspect existing embedded/external subtitles.
See the [Bazarr README](https://github.com/morpheus65535/bazarr). The
[OpenSubtitles migration note](https://wiki.bazarr.media/Troubleshooting/OpenSubtitles-migration/)
also confirms that the `.com` and `.org` accounts are separate.

## Preconditions

Agent:

```bash
kubectl get nodes -o wide
kubectl -n media get pvc media-nfs,bazarr-config
kubectl -n media get deployment radarr,sonarr
kubectl -n media get service radarr,sonarr
```

Required:

- the K3s node is `Ready`;
- `media-nfs` is `Bound`;
- Radarr and Sonarr are each `Available` and their ClusterIP Services have
  ready endpoints;
- `bazarr-config` does not already exist before the first deployment; and
- `bazarr.home.arpa` resolves to `192.168.30.103` through the existing
  wildcard record.

Owner:

1. Create or confirm a working OpenSubtitles.com account and store its
   credentials in the password manager.
2. Retrieve the current Radarr and Sonarr API keys from their K3s UIs. Do not
   commit, print, or put these values in a Kubernetes manifest.

STOP if either *arr service is unhealthy or its API key cannot be tested. This
service must not be pointed at a legacy Compose endpoint.

## Prepare reviewed manifests

Before deployment, resolve the then-current stable
`lscr.io/linuxserver/bazarr` release to an immutable manifest digest. Review
the release notes and record the version/digest in the implementation review.
Do not use a floating tag.

Create `infrastructure/node-main/10-bazarr/` with the same structure as
`06-radarr/` and `07-sonarr/`:

1. `01-config-persistentvolumeclaim.yaml` — `bazarr-config`, `5Gi`,
   `ReadWriteOnce`, `local-path`.
2. `02-deployment.yaml` — one `Recreate` replica with labels
   `app.kubernetes.io/name: bazarr` and
   `app.kubernetes.io/part-of: media-stack`; mount `bazarr-config` at
   `/config` and `media-nfs` at `/data`; use TCP startup/readiness probes on
   `6767`; do not force a Pod-level `runAsUser`.
3. `03-service.yaml` — ClusterIP Service `bazarr`, port `6767`.
4. `04-ingress.yaml` — Traefik Ingress for `bazarr.home.arpa` to the `bazarr`
   Service's HTTP port.
5. `kustomization.yaml` — list the four resources above.

No Kubernetes Secret is required for this initial deployment. Bazarr stores
the provider and *arr API credentials in its protected local `/config` state;
the values must never be copied into Git.

## Deploy

```bash
kubectl apply -f infrastructure/node-main/10-bazarr/01-config-persistentvolumeclaim.yaml
kubectl -n media get pvc bazarr-config
kubectl apply -f infrastructure/node-main/10-bazarr/02-deployment.yaml
kubectl -n media rollout status deployment/bazarr --timeout=300s
kubectl apply -f infrastructure/node-main/10-bazarr/03-service.yaml
kubectl -n media get endpointslice -l kubernetes.io/service-name=bazarr
kubectl apply -f infrastructure/node-main/10-bazarr/04-ingress.yaml
curl -fsS -o /dev/null http://bazarr.home.arpa
```

STOP if the PVC is not Bound after its Deployment consumer starts, the rollout
fails, there is no ready EndpointSlice, or the ingress does not return HTTP.

## First-run configuration and controlled acceptance

At `http://bazarr.home.arpa`:

1. Complete first-run setup and create the fresh administrator. Keep the
   password only in the password manager.
2. In the general/security settings, require Bazarr authentication for the
   LAN UI.
3. Add Radarr at `http://radarr:7878` with its existing API key. Save and run
   Bazarr's connection test. Repeat for Sonarr at `http://sonarr:8989`.
4. Create and assign the agreed English-only default language profile to both
   the movie and series collections. Enable embedded-subtitle detection so an
   existing embedded English track is not downloaded again.
5. Add OpenSubtitles.com as the only provider, enter its account credentials,
   save, and run the provider test. Do not add OpenSubtitles.org or a second
   provider during this step.
6. Enable automatic searches/downloads for newly added Radarr/Sonarr items.
   First perform a manual missing-subtitle search for one representative movie
   and one representative episode. Confirm that each selected subtitle is
   written beside the correct file below `/data/Video/Movies` or
   `/data/Video/TV Shows`, is readable by UID `1000`/GID `988`, and plays in a
   representative client.
7. After those two tests pass, start the full automatic search for the
   existing Radarr and Sonarr libraries. Allow Bazarr/OpenSubtitles to throttle
   requests; do not add providers or bypass rate limits to accelerate it.

## Validation

```bash
kubectl -n media get pod,service,ingress,pvc -l app.kubernetes.io/name=bazarr
kubectl -n media get events --sort-by=.lastTimestamp
kubectl -n media logs deployment/bazarr
```

Required acceptance:

- Bazarr, Service, Ingress, and `bazarr-config` are healthy and
  `http://bazarr.home.arpa` requires the new login.
- Both `http://radarr:7878` and `http://sonarr:8989` connection tests pass in
  Bazarr.
- The OpenSubtitles.com provider test passes and no OpenSubtitles.org provider
  is enabled.
- A controlled movie and episode each have the expected subtitle behavior;
  subtitles are not duplicated where embedded tracks satisfy the profile.
- Newly indexed test content triggers Bazarr automatically, and the full
  existing-library queue proceeds without repeated authentication or
  filesystem-permission errors.

Record the actual image version/digest, Bazarr URL, observed provider test,
and acceptance result in `infrastructure/node-main/system.md` **only after**
this step has passed.

## Rollback

There is no old Bazarr service to restart:

```bash
kubectl -n media scale deployment/bazarr --replicas=0
```

Preserve `bazarr-config` and all generated subtitle files for diagnosis. Do
not delete NAS data or the PVC automatically. Resume by scaling Bazarr back
up after correcting the configuration or provider issue.
