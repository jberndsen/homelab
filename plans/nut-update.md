# Fix the Proxmox NUT installation

Status: **known issue, unresolved** — updated 2026-10-15 after the outage
investigation and package review. No NUT fix was installed.

## Owner decision and preferred resolution

The owner chose to retain the distribution-managed NUT installation and track
this as a known issue. **The ideal fix is an official Debian package update
compatible with the Proxmox host**, demonstrably including the upstream
calibration-state fix. A locally built/managed package is not wanted: it would
add a recovery dependency that could be missed on a clean installation.

This is now an issue record and future validation guide, not an instruction to
build or install a custom package. Do not substitute an ad hoc binary, add
unstable repositories, or broaden this into a host upgrade. On a clean setup,
check the Debian package for the fix; installing the currently available stable
package reproduces the exposure. No permanent package hold was introduced.

Until a suitable official update is available, the existing shutdown protection
remains enabled, including its known false-shutdown risk. Increasing DEADTIME
alone does not fix stale calibration state. An outage can still result in early
shutdown if UPS communication is lost; this is not a rule to shut down merely
because the UPS reports on battery.

Preserve genuine low-battery/forced-shutdown protection, the secondary role,
and the Proxmox → guest shutdown sequence. The UPS communication interruption
is a separate unresolved issue. Deliberate mains removal, forced shutdown or
UPS power cycling requires an explicit maintenance window and backup readiness.

## Latest checks and work actually performed — 2026-10-15

- Proxmox still runs `nut-client 2.8.1-5`; `nut-common` is not installed as a
  separate package. `nut-monitor` is active. No NUT package, configuration,
  service state, UPS setting, network setting or VM setting was changed.
- Refreshed the host's configured APT metadata successfully. Installed and
  candidate `nut-client` remain `2.8.1-5` from Debian trixie/main. The configured
  trixie-updates/security repositories offered no replacement.
- Downloaded current Debian package indexes on the laptop: trixie-backports
  contained no `nut-client`; sid offered `2.8.5-1`. Its required `libnutscan4`
  depends on `libc6 >= 2.42`, so it is not a suitably narrow stable-host update.
  No sid repository was added and no sid package was installed. These are dated
  observations; recheck official repositories when revisiting the issue.
- Created `/root/nut-repair-20261015` with mode `0700` on Proxmox, copied
  `/etc/nut` to its `config-before` subdirectory, and downloaded the original
  `nut-client_2.8.1-5_amd64.deb` there. These protected host-local files are not
  a custom fix and contain configuration that must stay out of Git.
- No custom package was built; no isolated regression test or live outage test
  was run. The proposed build-environment inspection was interrupted before
  execution; the existing host shell was subsequently confirmed idle.
- VMs 102 and 103 were running and both guest-agent ping commands succeeded;
  VMs 100 and 101 remained stopped. Local `kubectl get nodes -o wide` confirmed
  K3s Ready with guest kernel `6.8.0-147-generic`. The full application recovery
  checks below are earlier investigation evidence, not newly repeated tests.
- `pvesm status` now reports `local`, `local-lvm` and `unas-pro` active.
  The NAS export can be listed and includes VM 103's
  `vzdump-qemu-103-2026_10_15-02_04_41.vma.zst` (32,703,632,526 bytes).
  No NAS permissions or storage configuration were changed in this session;
  why the earlier access denial cleared is unknown. Listing is not a restore
  test or confirmation of complete backup coverage.
- The host still routes directly to the UPS via `vmbr0.1`, source
  `192.168.1.39`. A kernel-journal query for 19:02–19:06 CEST in the incident
  boot showed VM 102's virtual interface leaving the bridge during shutdown;
  it did not establish the cause of the earlier UPS communication loss.

Read this together with repository-root `AGENTS.md`, the node/network system
records and the UPS section of repository-root `setup.MD`. The remaining sections preserve incident evidence
and describe verification required for a future official update.

## Diagnosis: what is established

**The host's NUT client initiated shutdown after losing UPS communication while
holding the calibration flag. This strongly matches the known stale-CAL defect
in the installed NUT 2.8.1-5 package.** It was not an immediate power cut to the
VM, and there is no logged low-battery or received forced-shutdown event in the
examined incident sequence.

Evidence came directly from Proxmox's previous-boot journal, redacted live
configuration, installed binary strings, and the Ubuntu guest journal.

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
| 19:42:30 | Current Proxmox boot began. |
| 19:42:46 | Current Ubuntu/K3s boot began. |

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

### Why this is a client defect, and limits of that conclusion

- Proxmox reports `nut-client 2.8.1-5`.
- `strings /lib/nut/upsmon` contained `calibration in progress` but neither
  `calibration finished` nor `NOTCAL`, consistent with missing upstream handling.
  Strings are supporting evidence; verify source/package patches when choosing
  the replacement.
- In upstream 2.8.1, `is_ups_critical()` returns critical when communication is
  lost and `ST_CAL` is set. This branch produces the exact observed message.
- The old parser can retain `ST_CAL` after the UPS stops reporting calibration.
  The upstream fix clears it and adds calibration-finished notification.
- **Increasing DEADTIME alone does not repair this branch.** It can declare
  critical immediately on communication loss with stale CAL, independently of
  the usual deadtime promotion of an on-battery UPS to low battery.
- Even a fixed client can legitimately shut down after sufficiently long lost
  communication while last known on battery. Fixing CAL does not guarantee
  survival of every network interruption.
- There was no packet capture or raw UPS-status history at the incident time.
  We cannot prove what every UPS status token was, whether calibration was
  genuinely active then, or why communication stopped. The stale-state diagnosis
  is strongly supported by the version, historical calibration and exact branch.

Primary references to recheck before selecting a package:

- [Upstream issue 2168: calibration state not cleared](https://github.com/networkupstools/nut/issues/2168)
- [Upstream fix PR 2169](https://github.com/networkupstools/nut/pull/2169)
- [NUT 2.8.1 upsmon source](https://raw.githubusercontent.com/networkupstools/nut/v2.8.1/clients/upsmon.c)
- [Debian report 1138630: same shutdown symptom on 2.8.1-5](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1138630)
- [NUT upsmon documentation](https://networkupstools.org/docs/man/upsmon.html)
- [NUT upsmon.conf documentation](https://networkupstools.org/docs/man/upsmon.conf.html)

The upstream PR was merged in November 2023 and assigned to 2.8.2. Verify the
actual downstream package carries the fix; do not choose solely by version label.
The initial inspection used cached APT metadata. The later refreshed check
also found installed and candidate both `2.8.1-5`; see the latest checks above.
No package installation was performed.

## Environment and access

- Repo: `/Users/jeroen/Develop/homelab`, public GitHub `jberndsen/homelab`, main.
  Deployable configuration belongs under `infrastructure/node-main`.
- Proxmox node name returned by `hostname`/journals is `proxmox`; repository
  calls the physical host `node-main`. Recorded PVE version is 9.2.4; verify
  live with `pveversion -v`. Current login banner showed kernel 7.0.14-23-pve.
- Host: `192.168.30.5`, SSH user `root`; alternate IP `192.168.1.39`.
- VM 103: `vm-ubuntu-k3s`, `192.168.30.103`, SSH user `jeroen`; key-based login
  worked. Passwordless sudo did not. Current guest kernel is 6.8.0-147-generic
  (the October 12 boot used 6.8.0-146-generic).
- VM 102: `vm-haos`. Both VMs have QEMU guest agents and `onboot: 1`.
- VMs 100 (`vm-us-media`) and 101 (`vm-us-immich`) were stopped. Do not start
  them merely because they appear in `qm list`; their intended state was not
  independently confirmed in this investigation.
- Proxmox root SSH rejected the agent's available key. The owner authenticated
  interactively in **herdr pane 2**, providing an existing root shell.
  At inspection the pane ID was `w2:p8`; IDs are ephemeral: rediscover first.
- For herdr, read `/Users/jeroen/.agents/skills/herdr/SKILL.md`, verify
  `HERDR_ENV=1`, run `herdr pane list`, then read the target pane before sending
  commands. Do not overwrite an in-progress command or password prompt.
  `herdr pane run <id> '<command>'` executes in the existing SSH session.
  Use `journalctl --no-pager` and `systemctl --no-pager -l` to avoid stuck pagers.
- A logged-in Proxmox browser session may be available as an alternative; the
  owner explicitly offered it. Discover current tabs rather than assuming IDs.
- UniFi has an authenticated Chrome session on `unifi.ui.com`, console
  **Unifi Cloud Gateway Max**. Use browser UI tools for read-only inspection.
  Do not dump its entire UPS panel: it can expose the NUT password in plain text.
- Use the local `kubectl` and preconfigured kubeconfig. Begin cluster work with
  `kubectl get nodes -o wide`. Sandbox network denials may require tool escalation;
  do not confuse them with actual host failure.
- Never put credentials, private keys, unredacted configs or raw sensitive
  support bundles in this public repo. The NUT password is on the host and in
  the owner's password manager, not in this handoff.

## Observed configuration

`/etc/nut/nut.conf`: `MODE=netclient`.

Active `/etc/nut/upsmon.conf` directives (secret removed):

```text
MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
POLLFREQ 5
POLLFREQALERT 5
HOSTSYNC 15
DEADTIME 15
POWERDOWNFLAG "/etc/killpower"
OFFDURATION 30
RBWARNTIME 43200
NOCOMMWARNTIME 300
FINALDELAY 5
MONITOR ups@192.168.1.34:3493 1 ups <redacted> secondary
FORCESSL 1
```

NUT monitor is enabled and currently running. TLS is used, but certificate
verification is disabled. Do not silently change authentication/TLS policy as
part of the calibration fix. Unauthenticated `upsc` is expected to be denied.

Network facts:

- `vmbr0` bridges `nic0`, has `192.168.30.5/24`, gateway `192.168.30.1`.
- `vmbr0.1` is tagged VLAN 1, has `192.168.1.39/24`.
- `ip route get 192.168.1.34` returned direct `dev vmbr0.1 src 192.168.1.39`.
  UPS monitoring therefore does not depend on gateway routing.
- UPS: UniFi UPS 2U `192.168.1.34`, NUT TCP 3493, ID `ups`, user `ups`.
- Proxmox and UPS share the UPS-powered Study switch. UniFi calls the Proxmox
  client `Intel NUC 2`, Study port 5: administratively Active, native VLAN 30,
  tagged VLANs Allow All. UPS is on port 2; NAS on port 3.
- UPS powers the host, switch and NAS according to the existing owner record.
  Actual power-cable/outlet placement was not physically inspected.
- Gateway is not UPS-powered. Its reboot is confirmed by uptime, but it is not
  proven to have caused this direct-link NUT interruption.
- VM traffic to NAS uses its gateway via `ens18`, unlike the host's direct VLAN
  interface. VLAN 30 internet traffic uses UniFi NordVPN with kill switch.

UniFi observations after the outage:

- Study switch uptime approximately 36 days; UPS uptime approximately 12 days.
- UPS Line mode, charging at 91%, output 233.6 V, estimated runtime 72 minutes
  at that later load. These are post-event readings, not incident-time telemetry.
- UPS firmware 1.6.4; NAS paired for safe shutdown, threshold five minutes of
  remaining runtime. `Power Cycle on Restore` unchecked.
- UPS event list had no event for this outage. Absence does not prove absence
  of a UPS service restart or status transition.
- NUC historical client last-connected date was October 2, so it is not reliable
  evidence of this incident's disconnect time.

## Recovery status and separate findings

Verified after restart:

- Proxmox: zero failed systemd units; `local` and `local-lvm` storage active.
- VMs 102 and 103 running; `qm guest cmd 102 ping` and `103 ping` succeeded.
- K3s node Ready; no memory, disk or PID pressure. Approximately 24% RAM used,
  1% CPU during one sample. Guest root filesystem 29% used, approximately 66 GB free.
- All eight media Deployments available, all running containers ready; restart
  counts stayed stable across checks. Startup probe warnings recovered.
- Argo applications `media` and `sealed-secrets` both Synced/Healthy.
- Nine media PVCs bound; both SealedSecrets report Synced=True.
- All web apps responded; redirects reached HTTP 200. Homepage health returned
  `up`, Jellyfin health `Healthy`. Transmission returned expected anonymous 401.
- Jellyfin could list `/data/Downloads`, `/data/Music`, `/data/Video`; NFS mount
  `192.168.1.32:/var/nfs/shared/media` was approximately 50% used.
- Full Jellyfin playback, authenticated Transmission RPC and HAOS application
  functionality were not tested. Do not describe these as verified.

### Earlier Proxmox NAS storage fault — currently accessible, cause unresolved

`pvesm status` reported `unas-pro` inactive with NFS **access denied by server**.
Its configured export is:

```text
server: 192.168.1.32
export: /volume/614eb849-21e8-49ba-84e8-dd36f30a5696/.srv/.unifi-drive/proxmox/.data
mountpoint: /mnt/pve/unas-pro
content: backup,images,snippets,vztmpl,rootdir,import,iso
```

Current host route to NAS is direct via `vmbr0.1`, source `192.168.1.39`.
**Hypothesis to check, not a finding:** export permissions might allow the old
Servers-VLAN source but not `.1.39`; alternatively the export path or NAS
configuration changed. Inspect actual exports and authorized clients before
changing anything. Do not widen to all clients or recreate storage blindly.
At the initial inspection, pvestatd logs repeated the denial. A narrow pre-outage 18:00–19:00 query
did not show the same error, but that is not proof the mount was healthy then.
At that inspection, backups targeting this storage could not work. The later
check found the storage active and backup files listable, without a storage
configuration change in this session. The cause of the denial and its recovery
remain unknown. Verify backup freshness,
coverage and accessibility before disruptive validation; a VM snapshot alone
is not an independent backup.

Other observations to assess without expanding scope unnecessarily:

- Current NUT startup logged a missing
  `/usr/lib/tmpfiles.d/nut-common-tmpfiles.conf`; service nevertheless started.
  Check package ownership/unit definitions during the package review.
- NUT initially reported no route to the UPS during boot, then established TLS
  and monitoring at 19:42:41. Inspect network ordering if recurrent; do not treat
  this recovered startup transient as proof of the original failure's cause.
- Kubelet shutdown grace periods are recorded as zero. VM shutdown was orderly,
  but that does not prove every stateful application received enough time to
  flush. Guest journal noted containerd shims remaining when K3s stopped.
  Treat application-aware shutdown hardening as a distinct follow-up.
- VM 103 took about 98 seconds for Proxmox to finish its shutdown task despite
  its journal ending after six seconds. No timeout failure was observed; do not
  infer corruption or force-stop without examining the task details.

## Future official-update procedure — not executed

### 1. Refresh baseline and preserve evidence

- Read repo/system instructions and inspect `git status` before changes.
- Reconfirm host/VM/cluster health and NUT connectivity. Collect package versions,
  package origins, systemd unit/drop-ins and redacted active config.
- Use incident timestamps or boot IDs, not blindly `-b -1` after further reboots.
  Incident host boot: `4bd281791e4f482b95092a449ea36bfd`.
  Incident guest boot: `f39417f343794df99682cbf0c30bdd1f`.
- Preserve relevant sanitized journal excerpts. Store full sensitive evidence
  and configuration backups only outside Git in an appropriately restricted
  location. Do not print the MONITOR password.
- Confirm current UPS line power and adequate battery before service maintenance.
- Resolve backup availability and understand the NAS mount fault before any
  disruptive test. Ask the owner only for facts/access that cannot be discovered.

Useful read-only commands on Proxmox:

```sh
date -Is
dpkg-query -W nut-client nut-common
apt-cache policy nut-client nut-common
systemctl cat nut-monitor.service --no-pager
systemctl status nut-monitor.service --no-pager -l
awk '!/^[[:space:]]*(#|$)/ {if ($1=="MONITOR") $5="<redacted>"; print}' /etc/nut/upsmon.conf
ip route get 192.168.1.34
ip route get 192.168.1.32
journalctl -b 4bd281791e4f482b95092a449ea36bfd -u nut-monitor --no-pager -o short-iso
qm list
qm guest cmd 102 ping
qm guest cmd 103 ping
```

`pvesm status` is a normal status command but can attempt to activate/mount
configured storage; during investigation it emitted the NAS mount rejection.

### 2. Select a supported fix before touching production

- Refresh package metadata when implementation is authorized and inspect current
  official Debian/Proxmox-compatible options. Prefer an official fixed package
  or security/stable update whose patches demonstrably include PR 2169.
- Review dependencies and an installation simulation. Avoid pulling an entire
  testing/unstable distribution into Proxmox to obtain one package.
- If no suitable official package exists, keep this tracked as a known issue.
  The owner explicitly declined a locally managed backport for clean-install
  recoverability. Do not build a custom replacement or mix unstable system
  dependencies into Proxmox to work around that decision.
- Verify the selected code clears CAL when absent and preserves genuine
  shutdown behavior. Compare the package source/patches, not just its name.
- Determine how future package updates will supersede the fix. Avoid a permanent
  hold that quietly blocks security maintenance. Do not implement unattended updates.

### 3. Validate the behavior in isolation

Do not reproduce the incident against the live UPS shutdown path.

Use an isolated NUT test environment with a simulated UPS/upsd and an inert
shutdown callback that records invocations. It must not connect to the live
UPS, shut down the host, or set its FSD flag. Build a meaningful regression
test covering the actual upsmon status parser and critical-state logic:

1. Send calibration status, then healthy line status without CAL.
2. Interrupt simulated communication while last known healthy: stale CAL must
   not trigger shutdown. Compare against the old implementation if practical.
3. Exercise real on-battery + low-battery and forced-shutdown states: expected
   shutdown callback must still occur under the configured secondary semantics.
4. Exercise on-battery communication loss separately and document the intended
   conservative behavior; do not accidentally test away legitimate protection.

Record inputs, timings and results. If no safe test harness is available, be
explicit about the remaining verification gap rather than forcing a live outage.

### 4. Install with rollback available

- Save protected copies of configs and identify a retrievable previous package.
  Review package scripts/config prompts and expected service restart behavior.
- Preserve secondary role, real shutdown command, power value, credentials and
  required TLS setting. Keep config ownership and permissions restrictive.
- Install only the selected necessary packages. Restart/reload only as required
  by the fix; acknowledge any brief monitoring gap. Keep an authenticated host
  console available and avoid doing this while on battery.
- Confirm the running binary really is the updated one. Check service startup,
  TLS authentication and fresh UPS status without leaking credentials.
- If service/authentication fails, restore compatible package/config state and
  verify monitoring is back. A rollback to 2.8.1-5 restores service but also
  restores the known defect; state that explicitly. Never leave protection off.

### 5. Investigate the lost UPS connection

- Correlate UPS, switch and host logs around 19:03–19:07 CEST. Inspect available
  UPS service logs, firmware history, switch port events/errors and host link logs.
- Verify direct VLAN 1 route and source address remain stable, no duplicate IP,
  no filtering/reprovisioning problem, and actual UPS/switch/NUC power cabling.
  Do not assume an IP conflict or wiring error without evidence.
- Look for UPS NUT-service/TLS session resets even if the UPS itself never rebooted.
- If existing evidence cannot establish the cause, propose scoped future
  monitoring with retention and redaction. Do not turn on indefinite verbose
  logging or capture credentials. Report the interruption cause as unresolved
  unless evidence resolves it.

### 6. Verify production and document recovery behavior

- Confirm NUT stays connected, no unexpected shutdown request, expected current
  UPS state, and package/service ownership consistency.
- Recheck both guest agents, VM running states, Proxmox local storage, NAS backup
  storage, cluster readiness, eight media Deployments, nine PVCs, Argo sync,
  SealedSecrets, app endpoints and NAS reads. Use the recovery checks in
  `infrastructure/node-main/ns-argo/SETUP.md`; its test manifests include writes
  and are not a read-only substitute.
- Validate cleared calibration state with the isolated test first. Observe a
  scheduled real self-test if feasible; manually triggering it requires awareness
  that the current production protection previously misbehaved.
- For a full physical outage test, agree a maintenance window, backup readiness,
  owner presence and recovery method. Do not run `upsmon -c fsd`, `upsdrvctl
  shutdown`, battery-exhaustion tests, outlet power cycles or unplug mains merely
  to demonstrate the patch.
- Investigate return-to-service separately: VM onboot works once Proxmox starts,
  but does not power on a halted physical NUC. Confirm BIOS power-recovery/WOL
  capabilities with the owner and assess the UPS's currently unchecked Power Cycle
  on Restore setting before proposing a design. Cycling UPS output also affects
  NAS/switch; never toggle it casually. Do not promise BIOS AC recovery will wake
  a gracefully halted host if AC was never actually removed.

### 7. Finish the handoff and repository changes

- Keep investigation history, package provenance, validation evidence and rollback
  details in this plan or focused operations documentation. Update `setup.MD` for
  a changed clean-install/recovery procedure. Update system records only for
  durable facts unavailable from code or normal read-only inspection, including
  owner decisions and any unresolved constraint future sessions must know.
  Remove or revise the short known-risk note once this issue is resolved.
- Preserve unrelated local work. At this handoff's creation, `TODO.md` (now
  `plans/TODO.md`),
  `infrastructure/node-main/system.md` and `infrastructure/network/system.md`
  already had uncommitted changes; do not overwrite or bundle them blindly.
- Keep credentials/private evidence out of commits. Owner authorizes task-related
  pushes to the public repo per AGENTS.md; review the exact staged diff first.
- Report separately: stale-CAL fix status, communication-loss cause, backup
  storage health, guest/application recovery, and physical restart behavior.

## Criteria for closing this known issue

- [ ] An official Debian package compatible with the host is installed and
  running, demonstrably includes the CAL-clearing fix, and is reproducible
  through the documented clean-install procedure.
- [ ] Regression test proves stale CAL does not cause false shutdown after return
  to line power, while real critical/FSD behavior remains intact.
- [ ] Production monitoring connected with preserved shutdown protection.
- [ ] Host, both active VMs and K3s/media recovery checks pass, with any untested
  application-level behavior explicitly stated.
- [ ] NAS backup storage is usable, or its unresolved failure is prominently
  tracked and disruptive testing is deferred until backup readiness is established.
- [ ] Communication-loss cause is evidenced or clearly left unresolved with a
  concrete diagnostic next step.
- [ ] Physical restart behavior is documented without claiming untested automatic
  power recovery.
- [ ] System records contain only relevant current facts/constraints; setup
  instructions, rollback and package maintenance strategy documented where
  appropriate; no secrets committed.
