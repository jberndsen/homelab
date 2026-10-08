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

Argo checkpoint 2 is complete: Sealed Secrets and independent encrypted NAS key
recovery are verified, and the owner confirmed password-manager storage.
Paused before checkpoint 3. Media remains manually managed, with automatic sync
disabled; adoption requires the Proxmox backup gate in the plan.
Git stores configuration; application data recovery depends
on Proxmox backups, and NAS media needs its own backup. Credentials and private
exports must stay out of this public repository.
