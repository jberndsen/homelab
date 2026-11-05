# Home Assistant owner decisions

- On 2026-10-22, the owner chose to stabilize and update the existing HAOS
  installation in Proxmox VM 102, superseding the earlier fresh-install proposal.
  Preserve its configuration and data; a reinstall is not part of the plan.
- The owner chose to retain HAOS on Servers VLAN 30.
  Keep device access and outbound routing validation separate from this
  placement decision; consult the network record for the VLAN's VPN policy.
- Initial scope is energy monitoring through a P1 meter and Enphase solar
  generation, plus control of Philips Hue lights.
- The owner confirmed on 2026-10-22 that the P1 reader is from HomeWizard,
  the solar gateway is an Enphase Envoy-S, and the Hue lights use a Hue Bridge.
- Future scope includes a Bosch washer and dryer and a home battery.
- The owner decided on 2026-10-22 to move the wired Hue Bridge to the IoT
  VLAN as a separate step after establishing a stable baseline. The move has not yet occurred;
  confirm network details and HA reachability before execution.
- The owner wants durable system records in this repository, following the
  same approach as the Kubernetes infrastructure records.
- On 2026-10-22 the owner confirmed saving the HA-native backup encryption key
  privately in their password manager. Keep the key out of Git and retain it
  with access to the corresponding backups for recovery.
- Stability is a priority. The owner wants device IP addresses stabilized
  step by step through UniFi and changes validated one at a time. Preserve
  investigation evidence for failures without treating hypotheses as facts.
- The owner confirmed on 2026-10-22 that they can access the Enphase cloud
  account using username/password, with no MFA enabled. HA was reconfigured
  using those credentials without manual token entry. Keep credentials private;
  revisit the authentication procedure if MFA is enabled later.

- On 2026-10-23, after reviewing the measured daily total and statistics
  rounding discrepancy, the owner chose to leave October 22 solar history
  unchanged for now. Consult the stability plan before any further correction.

- On 2026-10-23, the owner chose to connect HA to Prometheus as a separate,
  manual learning exercise using in-conversation teaching. The owner executes
  the live steps; agents should follow
  [the teaching plan](../../../plans/HA-prometheus-connection.md) rather than
  install an editor or deploy monitoring autonomously.

Record confirmed hardware constraints, owner decisions and recovery facts here.
Do not store credentials, duplicate discoverable runtime state, or treat Git as
a backup of Home Assistant's UI-managed configuration.

Host-level facts remain in [the node system record](../system.md).
Work is tracked in [the stability plan](../../../plans/home-assistant-stability.md).
