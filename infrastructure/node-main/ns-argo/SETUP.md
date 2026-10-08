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

## Checkpoint 2: controller and independent key recovery

Sealed Secrets is pinned to **v0.40.0**, reviewed on 2026-10-08 with its
[release notes](https://github.com/bitnami/sealed-secrets/releases/tag/v0.40.0)
and published advisories. This version includes fixes for
[the decryption oracle](https://github.com/bitnami/sealed-secrets/security/advisories/GHSA-qj4p-m373-p2wg)
and [scope widening](https://github.com/bitnami/sealed-secrets/security/advisories/GHSA-465p-v42x-3fmj).
The laptop uses matching `kubeseal v0.40.0` and `age v1.3.2`.

The manually applied `application-set.yaml` discovers
`infrastructure/node-main/ns-*/overlays/prod`. For example, segment 2 of
`infrastructure/node-main/ns-media/overlays/prod` is `ns-media`; trimming
`ns-` produces the Application name and namespace `media`. Sealed Secrets is
explicitly mapped to `kube-system`. Go templates fail on missing keys; Argo's
own installation is excluded and remains manually managed.

An ApplicationSet creates Applications; an Application renders Git and applies
resources only when synced. Both generated Applications have automatic sync,
pruning and self-healing disabled during adoption. Removing a discovered folder
preserves deployed resources: the set uses `preserveResourcesOnDeletion: true`
and generated Applications have no Argo deletion finalizer. The Sealed Secrets
CRD has `Prune=confirm,Delete=confirm`; deleting it would delete its custom
resources. No `CreateNamespace=true` is used: declare future app namespaces in
`cluster/namespace/`, register them in its Kustomization, and manually apply
`cluster/` before their first sync.

After reviewing, committing and pushing these files:

```sh
kubectl apply --dry-run=server -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl apply -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl -n argocd get applications,applicationsets
kubectl -n argocd get applications -o custom-columns='NAME:.metadata.name,NAMESPACE:.spec.destination.namespace,AUTO:.spec.syncPolicy.automated.enabled,FINALIZERS:.metadata.finalizers'
```

Expect exactly `media` → `media` and `sealed-secrets` → `kube-system`, both
with `AUTO=false` and no finalizers. Media's render error is expected until
checkpoint 3 replaces its ignored Secret files. **Do not sync media yet.**

In the Argo UI, open **sealed-secrets → Sync**, leave prune and force/replace
off, review the resource list, and sync only this Application. The equivalent
Kubernetes operation below requires the reviewed commit to be on GitHub; first
verify the Application has no active operation. `apply` selects ordinary apply,
and is not the destructive `replace` strategy.

```sh
git rev-parse HEAD
kubectl -n argocd get application sealed-secrets -o jsonpath='{.operation}{"\n"}'
# Replace REVIEWED_COMMIT with that pushed commit SHA.
kubectl -n argocd patch application sealed-secrets --type=merge -p \
  '{"operation":{"initiatedBy":{"username":"checkpoint-2"},"sync":{"revision":"REVIEWED_COMMIT","prune":false,"syncStrategy":{"apply":{}}}}}'
kubectl wait --for=condition=Established --timeout=60s crd/sealedsecrets.bitnami.com
kubectl -n kube-system rollout status deployment/sealed-secrets-controller --timeout=300s
kubectl -n argocd get application sealed-secrets
```

Separate Applications do not guarantee startup order. Wait for the controller
before creating any SealedSecrets. The upstream default renews sealing keys
every 30 days; old keys remain necessary for old ciphertext.

Install laptop tools with `brew install kubeseal age`. Key recovery uses
**smb://192.168.1.32/backups/NUC/kubernetes**, mounted at
**/Volumes/backups/NUC/kubernetes**, with exact backup/restore commands in the
adjacent NAS **README.md**. The owner keeps its age passphrase in their password
manager, outside Git and chat. Only encrypted `.yaml.age` backups, public
certificates, fingerprint inventories and recovery documentation belong there.
Keep private keys in protected local temporary files and clean them after use.

Before sealing, fetch the active certificate and compare its SHA-256 fingerprint
with the verified backup inventory. If missing, export and back up **all** key
Secrets using label `sealedsecrets.bitnami.com/sealed-secrets-key`, including
older keys. Encrypt locally with `age -p`, copy only ciphertext to NAS, decrypt
the NAS copy locally with `age -d`, and verify offline recovery of a harmless
test using `kubeseal --recovery-unseal --recovery-private-key`. Keep the previous
backup until the replacement is verified. Seal using that exact backed-up
public certificate with `kubeseal --cert`; this avoids renewal between checking
coverage and sealing. Adding a credential alone does not require another key
backup. No production key replacement is needed to test recovery.

## Continuing the rollout

Pause after checkpoint 2's controller and independent encrypted key backup are
verified. Media adoption and automatic reconciliation belong to checkpoints
3–4; TODO 1–3 remain open until all their checks pass.

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
