# Argo CD operations

Run commands from the repository root using the laptop's configured `kubectl`.
Read [node-main](../system.md) and [network](../../network/system.md) first.
The complete staged rollout and recovery requirements are in
[the agreed plan](../../../argocd-plan.md).

## Checkpoint 1: manual bootstrap

**Verified complete on 2026-10-08.** Argo v3.5.4 is running with all seven Pods
Ready and its three CRDs Established. HTTP UI, health and version checks passed.
The owner completed admin password rotation and replacement-password login;
the initial-password Secret is absent. Eight media Deployments remain available
and nine PVCs Bound, with Deployment specs/images and PVC/PV identities,
bindings and PV specs matching the private baseline. No Applications or
ApplicationSets exist. Resume at **Execution 3: Install Sealed Secrets and secure
recovery — checkpoint 2** in the plan, in a new session as requested by the owner.

This installs Argo CD only. No ApplicationSet or Applications are included, so
media is still managed manually. Do not adopt media until the later backup and
Sealed Secrets recovery gates in the plan have passed.

The pinned standard non-HA installation is **v3.5.4**, reviewed on 2026-10-08.
[Its release](https://github.com/argoproj/argo-cd/releases/tag/v3.5.4) fixes
critical vulnerabilities affecting the originally planned v3.5.3. The 3.5 line
is [tested with Kubernetes 1.36](https://argo-cd.readthedocs.io/en/stable/operator-manual/tested-kubernetes-versions/).
Upstream NetworkPolicies are retained. Upstream workloads have no resource
requests/limits; check actual usage after bootstrap.

1. Verify the node and review the cluster diff. Exit code 1 from `kubectl diff`
   means differences were found; an error has exit code greater than 1.

   ```sh
   kubectl get nodes -o wide
   kubectl kustomize infrastructure/node-main/cluster
   kubectl diff -k infrastructure/node-main/cluster
   kubectl apply -k infrastructure/node-main/cluster
   ```

   `cluster/` owns namespaces and the NAS PV manually. On an existing cluster,
   stop if the diff unexpectedly changes storage. A Namespace groups resources;
   declaring it here keeps its lifecycle outside Argo ownership.

2. Render and validate the pinned install, then apply it:

   ```sh
   kubectl kustomize infrastructure/node-main/ns-argo
   kubectl apply --server-side --dry-run=server -k infrastructure/node-main/ns-argo
   kubectl apply --server-side -k infrastructure/node-main/ns-argo
   ```

   Kustomize assembles upstream resources with our exact ConfigMap patch and
   Ingress. Server-side apply avoids storing the large CRD schemas in a
   last-applied annotation. Do not add force/replace options to resolve conflicts.
   Argo never manages this installation; changing these files requires a manual
   reapply. The ApplicationSet will be applied separately after its CRD exists.

3. Wait for the new API types and workloads:

   ```sh
   kubectl wait --for=condition=Established --timeout=60s \
     crd/applications.argoproj.io crd/applicationsets.argoproj.io crd/appprojects.argoproj.io
   kubectl -n argocd wait --for=condition=Available deployment --all --timeout=300s
   kubectl -n argocd rollout status statefulset/argocd-application-controller --timeout=300s
   kubectl -n argocd get pods,services,ingress
   kubectl top nodes
   kubectl -n argocd top pods
   ```

   A CRD registers a new Kubernetes resource type. A controller watches resources
   and repeatedly works toward their desired state. Healthy Pods show the
   controllers are running; an Application's eventual Synced/Healthy status will
   tell us whether its Git resources match the cluster and are working.

4. Open **http://argocd.home.arpa** on the trusted LAN. Traefik routes this host
   to `argocd-server` port 80; `server.insecure: "true"` disables Argo's own TLS.
   HTTP exposes credentials/session traffic to the network; restrict this to
   the trusted LAN as agreed. DNS should resolve to `192.168.30.103`.

   Retrieve the initial password in your own terminal (do not share its output):

   ```sh
   kubectl -n argocd get secret argocd-initial-admin-secret \
     -o jsonpath='{.data.password}' | base64 --decode; echo
   ```

   Log in as `admin`, open **User Info → Update Password**, and save the
   replacement in your password manager. Log out and verify login with the new
   password. Then remove the initial-password Secret:

   ```sh
   kubectl -n argocd delete secret argocd-initial-admin-secret
   ```

   The actual admin password hash lives in `argocd-secret`; deleting the initial
   Secret after changing the password removes the bootstrap copy. Never commit
   either Secret or put a password into a command argument/history.

   If you install the matching CLI, use an interactive password prompt:

   ```sh
   argocd login argocd.home.arpa:80 --plaintext --grpc-web --username admin
   ```

   `--insecure` alone still uses TLS. See the upstream
   [getting-started guide](https://argo-cd.readthedocs.io/en/stable/getting_started/)
   and [Ingress configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/).

## Continuing the rollout

Checkpoint 1 is complete; the owner requested a fresh session for the next stage.
The next stage installs Sealed
Secrets through an ApplicationSet with automatic sync disabled, and verifies an
independent encrypted key backup at `/Volumes/backups/NUC/kubernetes` before
sealing real credentials. Follow checkpoints 2–4 in the plan; TODO 1–3 remain
open until all their checks pass.

Before media adoption, the owner must confirm a fresh successful Proxmox backup
of VM 103 and that its included disks cover all application PVC data. The
private pre-install baseline is in ignored `plans/runtime/`; it is comparison
evidence, not a data backup. NAS media is external to the VM backup.

## Troubleshooting this stage

```sh
kubectl -n argocd get pods
kubectl -n argocd get events --sort-by=.lastTimestamp
kubectl -n argocd logs deployment/argocd-server --tail=50
kubectl -n argocd get configmap argocd-cmd-params-cm -o jsonpath='{.data.server\.insecure}'
curl --noproxy '*' -I http://argocd.home.arpa
```

An API timeout can be laptop VPN routing: verify LAN/VPN connectivity before
changing cluster configuration. For image pulls or Git access from the VM,
inspect UniFi Security logs for blocked registry, GitHub and API traffic.
If changing the HTTP ConfigMap after initial install, reapply then restart only
`deployment/argocd-server` and wait for its rollout.
