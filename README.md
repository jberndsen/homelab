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

Argo checkpoints 1–3 are complete. Media is adopted, both Applications are
Synced/Healthy, and storage identities, settings and credentials are unchanged.
The owner confirmed Homepage widgets and Jellyfin playback work. Automatic
sync, pruning and self-healing remain disabled; paused before checkpoint 4.
Encrypted NAS key recovery and the owner-confirmed VM 103 backup/PVC coverage
gate were verified on 2026-10-08. See the plan's progress section before continuing.
Git stores configuration; application data recovery depends
on Proxmox backups, and NAS media needs its own backup. Credentials and private
exports must stay out of this public repository.
