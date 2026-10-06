# Goal

Maintain the K3s media stack on `node-main`. Deployable Kubernetes
configuration belongs under `infrastructure/node-main`.

Consult the `infrastructure/**/system.md` records before making changes. They
hold current facts that are not represented in manifests. When node, workload,
or network information is unknown, do not guess: ask the owner and add the
confirmed fact to the appropriate system record.

Use the locally installed `kubectl` and its preconfigured kubeconfig for
cluster work. Start with `kubectl get nodes -o wide`.

For stateful image updates, review the release, take an application-native
backup, update one pinned digest, wait for rollout, and run the application's
smoke test. Do not introduce unattended updates or runtime socket access.

# Repository

The public repository is `https://github.com/jberndsen/homelab`, with default
branch `main`. Local `origin` uses `git@github.com:jberndsen/homelab.git`.
The owner authorizes pushing task-related changes to this repository. Keep
plaintext credentials, private keys, and local reference files out of commits.
Update the relevant system records whenever confirmed infrastructure facts change.