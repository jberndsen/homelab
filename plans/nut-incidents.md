# NUT incident evidence — October 2026

Historical evidence, not current installation instructions. The stale-CAL fix
was installed on 2026-11-05; see [execution record](nut-update.md) and
[NUT operations](../infrastructure/node-main/nut/README.md).
These observations do not establish current service health or prove the cause
of communication loss. Raw UPS status at each failure was not captured.

## October 15

The host journal identified NUT as the shutdown initiator. Installed version
was `2.8.1-5`; its parser did not clear CAL. The exact message was
`was last known to be calibrating and currently is not communicating, assuming dead`.
No low-battery or received FSD event was logged in the examined sequence.
Host incident boot: `4bd281791e4f482b95092a449ea36bfd`.
Guest incident boot: `f39417f343794df99682cbf0c30bdd1f`.

### Timeline (Europe/Amsterdam, CEST / UTC+02:00)

| Time | Evidence |
| --- | --- |
| Oct 6, 03:52:23 | Same long-running `upsmon` process logged on battery and calibration in progress. |
| Oct 6, 03:52:33 | UPS returned to line power. |
| Oct 13, 03:52:23–33 | Another brief on-battery/line transition; no new calibration-start notification in the filtered history. |
| Oct 15, approximately 19:03 | Owner reported outage beginning, lasting a few minutes. |
| 19:03:53 | NUT logged UPS on battery. |
| 19:05:17 | `Poll UPS [ups@192.168.1.34:3493] failed -` (no useful reason followed the dash). |
| 19:05:17 | `Communications with UPS ups@192.168.1.34:3493 lost`. |
| 19:05:17 | `UPS [ups@192.168.1.34:3493] was last known to be calibrating and currently is not communicating, assuming dead`. |
| 19:05:17 | `Executing automatic power-fail shutdown`, then `Auto logout and shutdown proceeding`. |
| 19:05:22 | Five-second FINALDELAY expired; `/sbin/shutdown -h +0` initiated host shutdown. |
| 19:05:24 | Proxmox requested shutdown of VMs 103 and 102, each with a 180-second timeout. |
| 19:05:25 | VM 103 QEMU agent logged `guest-shutdown called`; logind said `hypervisor initiated shutdown`. |
| 19:05:31 | VM 103 journal reached poweroff, filesystem sync and final SIGTERM. This is the journal endpoint, not proof that the QEMU process exited at this instant. |
| 19:05:38 | Proxmox completed VM 102 shutdown task. |
| 19:07:02 | Proxmox completed VM 103 shutdown task. |
| 19:07:03 | Proxmox logged all VMs/CTs stopped and stopall task `OK`. |
| 19:07:19 | Host journal reached final poweroff and filesystem sync. |
| Later | Owner found NUC power LED off. |
| 19:42:30 | Recovery Proxmox boot began. |
| 19:42:46 | Recovery Ubuntu/K3s boot began. |

Guest journal timestamps use UTC: subtract two hours from the above. Use explicit
timezone arguments when comparing journals. The owner did not provide an exact
power-restoration time or confirm the physical power-on action; do not invent it.

### Initial reachability inspection — historical evidence

Read-only checks around 19:38–19:45 CEST on October 15 found the gateway,
NAS SMB and UPS NUT TCP port reachable. Gateway DNS still resolved
`argocd.home.arpa` to `192.168.30.103`; both laptop VPNs were disconnected.
UniFi reported all three switches, all three APs, gateway, UPS and NAS online.
Gateway uptime was approximately 34 minutes, versus 36 days for the Study
switch and 12 days for the UPS. These observations do not establish the cause
of the NUT connection loss.

Proxmox SSH at `192.168.30.5` and `192.168.1.39`, HTTPS at
`192.168.30.5:8006`, and K3s API/HTTP at `192.168.30.103` timed out.
ARP for `192.168.1.39` was incomplete; secondary host `192.168.1.33:22` also
timed out. Host-level checks were initially blocked. Later journal and owner
evidence established the orderly host shutdown described above. These failed
connections are incident history, not declarations of current host availability.

### Earlier controlled VM reboot — distinct from UPS validation

On October 12, after the owner's package upgrade, VM 103 was rebooted with
an explicit drain: cordon, gracefully drain seven stateful media workloads
with a 120-second termination allowance, sync, reboot and uncordon. Homepage
and system Pods were not drained, preserving temporary volumes. Eight media
Deployments returned Available, both Argo Applications Synced/Healthy, all
nine PVC/PV bindings were unchanged and Jellyfin could read the NAS mount.
The guest then ran kernel `6.8.0-146-generic`; the later October 15 boot ran
`6.8.0-147-generic`. This controlled procedure did not validate unattended UPS
shutdown. Kubelet shutdown grace periods were recorded as zero; inspect live
configuration before designing application-aware shutdown hardening.


## Recurrence — 2026-10-24

Read-only host journal inspection confirmed the same shutdown trigger with
`nut-client 2.8.1-5` still installed. No live configuration or package changes
were made, and replacement-package availability was not reassessed.

All times are Europe/Amsterdam, CEST (UTC+02:00):

- October 20, 03:52:23: `upsmon[913]` reported on battery and calibration in
  progress; at 03:52:33 the same process reported line power.
- October 24, 03:16:21: UPS polling failed and communications were lost.
  NUT immediately logged `was last known to be calibrating and currently is
  not communicating, assuming dead`, followed by
  `Executing automatic power-fail shutdown`.
- 03:16:26: host shutdown scheduled. At 03:16:28 Proxmox requested shutdown
  of VMs 102 and 103, each with a 180-second timeout. Their tasks ended at
  03:16:38 and 03:18:03 respectively; `stopall` completed `OK` at 03:18:04.
- 03:18:11: host reached poweroff target and filesystem sync.
- 21:17:23: next host boot began. NUT reconnected at 21:17:34 after a
  transient startup no-route error. The physical restart action is unconfirmed.

Unlike October 15, no fresh on-battery event preceded this shutdown in the
examined NUT sequence; neither low battery nor received forced shutdown was
logged. This strengthens the stale-CAL diagnosis: communication loss triggered
shutdown days after calibration and return to line power. Raw UPS status at
the failure was not captured, so actual calibration state is not proven.

The complete host journal from 03:14:00 to 03:16:26 contained only the NUT
trigger entries; the kernel journal from 03:00:00 to 03:16:26 had no entries.
The cause of communication loss remains unknown. Correlate UPS service/restart
and switch events at 03:16:21; no UPS-side or switch logs were obtained here.

For future journal queries, incident boot is
`65d16281589e4f4e8d5681a06f1a4f77`; recovery boot is
`57cc752f7bbd4652990b67322fbd019e`.

Recovery checks found both VMs running and guest-agent pings successful, no
failed host units, UPS status `OL`, and the unchanged direct UPS route through
`vmbr0.1` from `192.168.1.39`. K3s was Ready, all eight media Deployments were
available, and Argo `media`, `monitoring`, and `sealed-secrets` were
Synced/Healthy. Playback, HA functionality and backup restore were not tested.


## October 29

Read-only investigation on October 29 established the following. Times below
are Europe/Amsterdam **CET (UTC+01:00)**, unless otherwise stated.

- Installed `nut-client` and cached APT candidate: `2.8.1-5`. Debian's stable
  package page also listed that version. APT metadata was not refreshed during
  this investigation; current backports availability was not established.
- October 27, 02:52:27: the same `upsmon` process reported on battery and
  calibration. At 02:52:37 it reported line power.
- October 29, 03:18:05: polling failed and communication was lost. NUT logged
  `was last known to be calibrating and currently is not communicating,
  assuming dead`, then `Executing automatic power-fail shutdown`.
- 03:18:10: host shutdown scheduled; 03:18:12: Proxmox started shutting down
  VMs 102 and 103. Tasks ended at 03:18:30 and 03:19:21 respectively;
  `stopall` completed `OK` at 03:19:22. Host reached poweroff and filesystem
  sync at 03:19:28.
- Next host boot began at 21:05:19. How physical power-on occurred is unknown.
- Incident boot: `57cc752f7bbd4652990b67322fbd019e`.
  Recovery boot: `cc635455193c4814bcd08f2f70ddfd86`.
- No fresh on-battery, low-battery or received forced-shutdown event appeared
  immediately before shutdown in the examined NUT sequence. The kernel journal
  from 03:00:00 through 03:18:09 had no entries. Surrounding host logs did not
  establish the communication-loss cause.
- Earlier matching shutdowns: October 15 and October 24 (those records use
  CEST, UTC+02:00). Calendar intervals shortened from nine days to five days.
  Three incidents justify increased priority, not a proven accelerating trend.
- Recovery inspection found both VMs running, no failed host units, UPS status
  `OL`, the direct UPS route intact, K3s Ready and Argo `media`, `monitoring`
  and `sealed-secrets` Synced/Healthy. This did not test playback, HA functions
  or backup restoration.

NUT initiating shutdown is confirmed; stale CAL is strongly supported by the
exact message, package version and earlier calibration/line-power sequence.
Raw UPS status at failure was not captured. Do not claim measured battery
health at the incident or proof of the UPS's actual calibration state then.

This fix addresses stale CAL, not every possible shutdown on communication
loss. Existing conservative behavior after lost communication while genuinely
on battery remains. UPS communication-loss cause and automatic physical restart
remain separate unresolved issues, not blockers or extra projects for this fix.


## Separate unresolved findings

- During the October 15 inspection, NAS backup storage reported NFS access
  denied, then became accessible without a configuration change. The cause of
  failure and recovery was not established. Active storage or a listed archive
  is not a restore test; see the [recovery guidance](../infrastructure/node-main/ns-argo/SETUP.md#recovery-checks).
- The October 15 UPS panel had no event for the outage. Switch uptime and UPS
  uptime were longer than gateway uptime, but the host's direct VLAN 1 UPS route
  did not traverse that gateway. These readings did not identify the fault.
  Proxmox was on Study switch port 5, UPS port 2 and NAS port 3 during inspection;
  recheck live topology before using these as diagnostic inputs.
- The UPS had NAS safe shutdown set to five minutes remaining runtime and
  Power Cycle on Restore unchecked during that inspection. These are historical
  UI observations, not instructions to change either setting.
- Physical restart actions after these incidents were unconfirmed. VM onboot
  does not power on a halted physical host; BIOS AC recovery cannot be assumed
  to work unless AC actually disappears and returns.
- The old service's missing tmpfiles warning and recovered boot-time no-route
  error were distinct from the later communication-loss trigger.

## Superseded decisions

On October 15 the owner preferred waiting for an official compatible package.
On October 29 the owner allowed a narrowly patched custom Debian package with
published source/binary/checksum and a temporary hold if no official fix existed,
and explicitly declined a simulated UPS regression test. The November 5 official
package made that build/release/hold fallback unnecessary. Neither waiting for a
custom build nor simulated failure testing is a remaining implementation step.
