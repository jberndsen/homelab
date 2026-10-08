# Homelab

K3s media stack on `node-main`, with deployable configuration under
`infrastructure/node-main`.

Argo CD reconciles media and Sealed Secrets from public GitHub `main`.
Namespaces, the NAS PV, Argo itself and its ApplicationSet are manually managed.
Git stores configuration; application data, NAS media and sealing keys require
separate recovery sources. Keep credentials and private keys outside Git.

- [Node and storage facts](infrastructure/node-main/system.md)
- [Network facts](infrastructure/network/system.md)
- [Secondary node](infrastructure/node-secondary/system.md)
- [Infrastructure recovery](setup.MD)
- [Argo CD bootstrap, operations and recovery](infrastructure/node-main/ns-argo/SETUP.md)
- [Next steps](TODO.md)
