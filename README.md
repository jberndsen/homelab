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

Argo rollout is in progress. Media is still managed manually until the adoption
checkpoint passes. Git stores configuration; application data recovery depends
on Proxmox backups, and NAS media needs its own backup. Credentials and private
exports must stay out of this public repository.
