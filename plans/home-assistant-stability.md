# Home Assistant stability and follow-ups

Status: **updates and integration repairs completed; validation and follow-ups remain**.
Latest recorded HA inspection: 2026-10-23. Reinspect live state before acting;
this document is not evidence of current versions, backup retention or health.

Retain HAOS VM 102, its data and Servers VLAN 30 placement. Do not reinstall.
Owner decisions are in [the HA system record](../infrastructure/node-main/home-assistant/system.md);
update/recovery procedures are in [HA operations](../infrastructure/node-main/home-assistant/README.md).
Monitoring remains a separate [owner-executed teaching plan](HA-prometheus-connection.md).
This document does not authorize running that lesson or changing production.

## Completed work — October 22–23, 2026

- Stabilized Envoy access: UniFi showed the device at `192.168.1.246` while HA
  targeted `.117`. The owner retained `.246`, updated the reservation and
  reconfigured HA with Enphase cloud credentials. Address and authentication
  changed together, so the original API 500 cause was not isolated.
- Owner confirmed UniFi reservations for Envoy `192.168.1.246`, P1
  `192.168.50.27`, Hue Bridge `192.168.1.128` and HAOS `192.168.30.166`.
  Recheck actual leases/reservations before network work; renewal stability
  was not tested. Do not infer the IoT VLAN ID or firewall policy from `.50`.
- Core updated `2026.3.2` → `2026.10.2`, with About-page verification.
- HomeWizard authentication migration completed October 23 around 20:36.
  Existing device/config-entry identity and import/export entity IDs remained;
  entity count increased 34 → 39. Live power refreshed, import continued from
  4,370.67 to 4,370.75 kWh, export stayed 6,265.27 kWh, and the brief unavailable
  interval recovered. Repairs showed none pending. Do not repeat the migration.
- Hue on/off, brightness and app state feedback passed owner checks.
- Samsung sign-in and TV recovery completed: owner saw The Frame in the phone
  app and HA and successfully turned it off through HA. Further TV diagnosis is
  needed only if symptoms recur. Earlier SSE `ClientPayloadError` /
  `TransferEncodingError` and token-refresh failures were distinct observations,
  not proof of a VPN or cloud root cause. No TV network/reset/pairing change was made.
- HAOS updated `17.1` → stable `18.3` after a fresh VM102-only NAS backup.
  The owner rebooted; refreshed About confirmed OS 18.3, Core 2026.10.2,
  Supervisor 2026.10.3 and frontend 20260930.4. Earlier deferral and pre-reboot
  version mismatch are resolved; do not reinstall 18.3 from this historical plan.
- VM hardware inspection showed VirtIO NIC/SCSI, q35/UEFI, 4 GiB RAM, 2 vCPU
  and no physical device pass-through. No VM/VLAN configuration changed.
- October 23 solar total 3.21 kWh agreed with Envoy daily 3.211 kWh,
  recorder lifetime delta 3.213 kWh and owner-reported Enphase 3.2 kWh at display
  precision. No negative October 23 day-boundary discontinuity was observed.

## Remaining work

1. Complete the **post-HAOS-update smoke check**, which was not recorded:
   P1 power freshness/counter continuity, Envoy daily/current production, Hue
   control and feedback, Repairs and logs since reboot. Check the next overnight
   boundary after P1 migration. Do not report these as already tested.
2. Investigate P1 power-failure diagnostics separately, as requested by the owner.
   October 23 counters were 8 power failures and 7 long failures, unchanged by
   authentication migration; voltage sags by phase were 2/1/2 and swells 2/2/3.
   Establish counter semantics and available timestamps, observe increments and
   correlate with independent UPS/outage evidence. Counts alone do not date
   events or establish a cause of HA instability.
3. Keep October 22 solar at the owner-accepted **3.780 kWh**. No further history
   correction is authorized. Preserve the evidence below if reconsidered later.
4. Plan the Hue IoT move below only after confirming the stable baseline and
   actual network/port details. Keep HAOS on Servers VLAN 30.
5. Establish a durable off-VM native-backup destination/schedule and an isolated
   restore test. Native scheduling was Never at inspection; nightly Proxmox
   backup exists but does not prove application restore. Recheck current policy.
6. Resume the separate Prometheus teaching plan at its first incomplete step;
   no editor, metrics credential, scrape job or dashboard was deployed by this work.

## Recovery evidence — dated, verify availability before use

The owner saved the native backup encryption key in their password manager.
Downloaded native archives can be decrypted even when the server copy is encrypted;
keep them private under ignored `tmp/` with mode 0600. Do not commit them or send
backups to malware scanners. Readability checks are not restore tests.

| Recovery point | Coverage and verification |
| --- | --- |
| Native `f8a2769b`, Oct 22 21:44 | Before Core update; Core 2026.3.2; settings/history, media/share/SSL, no installed apps. Download `tmp/automatic_backup_2026_3_2.tar`, 57,569,280 bytes; all nested archives readable. |
| Native `09bb68ea`, Oct 23 20:28 | Core 2026.10.2, **before P1 migration**. Download `tmp/automatic_backup_2026_10_2.tar`, 56,791,040 bytes; four nested archives readable; recorder quick_check passed. |
| Native `ac35f78b`, Oct 23 20:56 | Post-P1 migration, pre-HAOS update; 54.27 MB, encrypted inside HAOS. No separately downloaded copy verified. |
| VM102 NAS backup, Oct 23 21:03:17–21:04:00 | `vzdump-qemu-102-2026_10_23-21_03_17.vma.zst` on unas-pro, 3.42 GB; 32 GiB SCSI + 4 MiB EFI, guest freeze/thaw, TASK OK, no pruning. Contains post-P1 configuration and native ac35f78b. Restore untested. |

Native download SHA-256:

- `f8a2769b`: `86e84449e35256cdc9d0901376ee1e3e501ae81b44582ff199e9ce19aef06bee`
- `09bb68ea`: `12d2357d57c6ea4e2024ed046ce608d8929be73d2337782e8f972b46e8dacf9e`

The owner explicitly stopped Chrome download troubleshooting and accepted the
fresh whole-VM NAS backup as the HAOS update recovery point. No Chrome security
setting was changed and compromise was not established. Do not resume that
browser investigation from old session notes. A future change needs a newly
verified recovery point, not an assumption these archives are still retained.

## Solar discontinuity and correction — October 22

The pre-update recorder already reproduced -749.397 kWh, so the discontinuity
predated the Core update. Last old hourly point May 9 01:00 UTC had lifetime
state 6.081965 MWh, sum 0.753056 MWh; October 22 09:00 UTC had state
8.543494 MWh, sum 0.000177 MWh. The cause of the reset remains unknown.

One owner-approved Statistics → Adjust sum operation was submitted for
October 22 11:00–12:00 Europe/Amsterdam on
`sensor.envoy_482332031113_lifetime_energy_production`, entering change
0.000177 MWh. Dashboard solar changed from -749.4 to 3.78 kWh. No statistics
were deleted and no second adjustment was submitted. The owner chose to keep
that result after the precision discrepancy was explained on October 23.

Private read-only investigation copies were `tmp/ha-energy-baseline.db` and
`tmp/ha-energy-2026-10-23.db`; verify existence before use. Keep SQLite inspection
read-only/immutable and `TMPDIR` inside repository `tmp/`.

## Solar history evidence — preserve the accepted correction

Both backup databases inspected read-only; historical lifetime state values
are identical. Only cumulative sums changed by a constant **0.753177 MWh**.

| Quantity | Exact value |
| --- | --- |
| Last old baseline (May 9) lifetime sum | 0.753056 MWh |
| October 22 11:00 hourly sum, before / after adjustment | 0.000177 / 0.753354 MWh |
| October 22 final lifetime sum, before / after | 0.003659 / 0.756836 MWh |
| Resulting October 22 dashboard total | 3.780 kWh |
| Exact offset needed solely to remove old discontinuity | 0.753056 MWh |
| Excess offset applied | 0.000121 MWh = 0.121 kWh |
| Envoy October 22 daily sensor final state | 4.049 kWh |
| Envoy daily state when collection resumed at 11:35 | 0.388 kWh |
| Daily sensor recorded growth after resumption | 3.661 kWh |

The applied offset equals `0.000177 - (-0.753)` MWh, consistent with the
rounded old period value being used. This arithmetic establishes the mismatch;
the exact frontend mechanism is now confirmed from the installed tag
[20260930.4 source](https://github.com/home-assistant/frontend/blob/20260930.4/src/panels/config/tools/statistics/dialog-statistics-adjust-sum.ts):
`_setChosenStatistic` rounds change to entity display precision (minimum 2),
stores it as `_origAmount`, then `_fixIssue` submits `_amount - _origAmount`.
Thus selected -0.753 MWh plus entered 0.000177 produces +0.753177 MWh,
matching the database exactly. UI selection of a small 0 MWh change can also
round it to zero. Any future adjustment must account for this behavior.
The earlier 3.659 kWh expected value was the lifetime growth since recovery,
not complete-day production. The daily sensor already held 0.388 kWh at first
sample, explaining most of the missing early production; lifetime versus daily
polling differs by 0.002 kWh. Enphase cloud owner total 4.0 kWh agrees with
4.049 kWh at its display precision. Do not invent missing months or silently
backfill the day's early production. No second adjustment was submitted.
Statistics UI showed rounded 0 MWh, including New value 0; closed unchanged.


## Deliberate Hue Bridge move to IoT

- Inspect the actual IoT VLAN ID, subnet, DHCP scope and firewall policy;
  HAOS will remain on Servers VLAN 30. Do not infer VLAN ID from `.50`.
- Identify the Bridge's MAC and physical switch port, including any shared
  downstream equipment. Capture the current port configuration and reservation
  for rollback; do not change a shared uplink as if it were a dedicated port.
- Select a conflict-free IoT reservation and define the required HA-to-Bridge
  access. Review discovery needs separately from direct IP connectivity; avoid
  broad inter-VLAN access rules merely to make discovery work.
- Move the dedicated Bridge port to the intended IoT untagged/access network
  and arrange its DHCP reservation/lease renewal. Verify its actual new address
  in UniFi before changing the HA integration endpoint as needed.
- Verify HA light on/off, brightness and state feedback, plus Hue app access
  from intended client networks. Observe operation across reconnect/reload.
- If validation fails, restore the original port configuration and reservation,
  renew the lease and restore the old HA endpoint as applicable. Confirm working
  light control before attempting further changes.



## Release-review history

The Core update review covered April–October 2026 breaking changes and the
installed integrations; no custom integrations or automation/script dependencies
were reported. HAOS review covered 17.2 and 18.0–18.3, with 18.4.rc1 considered
only as evidence of known fixes, not an installation target. The owner later
approved stable 18.3 after initially deferring it. Future updates require a fresh
review of intervening releases and a recovery point; these old targets are not
standing upgrade instructions.
