# Step 9: Watchtower replacement and future network improvement

No Kubernetes YAML is applied in this decision step.

## Initial update policy

Do not migrate Watchtower or grant any workload a runtime socket. The old
Watchtower is already stopped. For each image update:

1. Read upstream release notes and confirm backup compatibility.
2. Resolve and review one immutable registry manifest digest.
3. For a stateful app, create/download an application-native backup first.
4. Change only that app's digest in `plans/manifests/`.
5. Apply its manifest and wait for rollout status.
6. Inspect Pod logs/events and repeat that app's acceptance test.
7. Retain the previous digest and backup. `rollout undo` does not reverse an
   application database migration.

Never enable unattended stateful-image updates until backup/restore and
rollback have been rehearsed.

## Deferred Transmission-only VPN identity

The initial design intentionally routes every K3s workload through VLAN 30's
NordVPN policy because UniFi sees normal Flannel egress as the VM source.
After migration parity, investigate a dedicated, UniFi-selectable Transmission
identity. Candidate designs include a second VM/interface or a deliberately
designed Multus attachment. This is not part of the current manifests and must
not be added without documenting Proxmox VLAN topology, addressing, routing,
DNS/NFS allowances, and fail-closed tests.
