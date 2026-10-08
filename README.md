# Homelab

K3s media stack on `node-main`, with deployable configuration under
`infrastructure/node-main`.

- [System and node facts](infrastructure/node-main/system.md)
- [Network facts](infrastructure/network/system.md)
- [Infrastructure recovery](setup.MD)
- [Argo CD bootstrap and operations](infrastructure/node-main/ns-argo/SETUP.md)
- [Staged Argo CD and Sealed Secrets plan](argocd-plan.md)
- [Next steps](TODO.md)
- [Argo bootstrap lesson](lessons/0001-argocd-bootstrap.html)
- [Sealed Secrets recovery lesson](lessons/0002-sealed-secrets-recovery.html)

Argo checkpoints 1–4 and Execution 6's documentation audit are complete.
Media and Sealed Secrets are Synced/Healthy; automatic sync, pruning and
self-healing remain enabled. Storage identities, settings and credentials are
unchanged, and all deletion safeguards remain intact. The disposable
reconciliation test is gone. The owner confirmed Homepage widgets and Jellyfin
playback at checkpoint 3; read-only checks passed again in Execution 6.

For recovery, choose [a fresh cluster, ordinary VM restore or guarded restore](infrastructure/node-main/ns-argo/SETUP.md#choose-the-recovery-path)
before starting. Fresh clusters must use the disabled ApplicationSet copy until
keys, data, storage and smoke checks pass. Ordinary VM restores may resume Argo
immediately. Git stores configuration; VM backups restore application data;
NAS media and sealing keys need separate backups. The recorded VM backup and
independent encrypted key-recovery evidence are in the operations guide.
VM restore, NAS-media recovery and off-host backup copies were not verified by
this audit. Credentials, private keys and local reference files stay outside Git.
TODO 1–3 are complete; later TODOs remain outside this work.
