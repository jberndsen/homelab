# NUT stale-calibration fix — execution record

Status: **deployed 2026-11-05** using official Debian
`nut-client 2.8.1-5+deb13u1` on Proxmox (trixie, amd64).
The authoritative clean-install, update, verification and rollback procedure is
[NUT operations](../infrastructure/node-main/nut/README.md).
[October incident evidence](nut-incidents.md) is retained separately as history.
This record replaces the old waiting plan and the October 29 custom-package
handoff (`nut-fix.md`); neither remains an active implementation plan.

## Official-package decision

Debian accepted the update on November 4 into **trixie-proposed-updates**.
Refreshed regular host repositories still offered `2.8.1-5` on November 5.
The official stable-targeted package avoids the custom build/release/hold
fallback. Proposed-updates is a staging suite, not a stable point release yet.
A scoped source and APT pin admit only the reviewed exact `nut-client` version;
all other proposed packages have priority -1. Newer normal stable/security
packages can supersede it through regular reviewed updates. No unstable source,
distribution upgrade, custom binary, permanent hold or unattended update was added.

Provenance:

- [Debian bug/upload](https://bugs.debian.org/1138630)
- [Upstream CAL fix](https://github.com/networkupstools/nut/pull/2169)
- [Official binary](https://deb.debian.org/debian/pool/main/n/nut/nut-client_2.8.1-5+deb13u1_amd64.deb)
- [Official Debian source changes](https://deb.debian.org/debian/pool/main/n/nut/nut_2.8.1-5+deb13u1.debian.tar.xz)
- [Source descriptor](https://deb.debian.org/debian/pool/main/n/nut/nut_2.8.1-5+deb13u1.dsc)

SHA-256 values reviewed:

| Artifact | SHA-256 |
| --- | --- |
| Official amd64 `.deb` | `feb6fdec2df70d7cfd603d2e2058bce2e0a3b1fceb24b5b08e1b41a39ab9a301` |
| Debian source changes `.tar.xz` | `506804e1f74c5e75f786d9f2fdd82709e9042a5af39ec0c59b3fbfcc2865f065` |
| Packaged and running `/usr/lib/nut/upsmon` | `66983df8f19be341d989c6d53c1e1fb593757e98b142bc7d3880d00af33d5b6f` |

APT validated the signed Debian repository metadata and downloaded the package.
Source comparison against `2.8.1-5` showed only changelog, patch-series entry
and `upsmon-clear-CAL-flag.patch`. That patch adds two lines in `parse_status()`
to clear `ST_CAL` when `CAL` is absent. Critical/FSD logic is unchanged. The patch
is deliberately narrower than upstream's accompanying notification additions:
`NOTCAL` and “calibration finished” are not valid binary-string checks for it.
Package dependencies and maintainer restart behavior were reviewed.

## Executed steps and verification

All observations below are from November 5, 2026, Europe/Amsterdam CET.
They are deployment evidence, not a claim about indefinite future health.

1. Read system records and both NUT plans; inspected and preserved unrelated
   working-tree changes. Began cluster inspection with `kubectl get nodes -o wide`.
2. Inspected Proxmox packages, holds, service, route, guests and redacted config;
   refreshed APT and checked official archive/source availability.
3. Confirmed UPS `OL`, charge 100%, runtime 4,340 seconds before maintenance.
   Verified the existing root-protected `/root/nut-repair-20261015` configuration
   backup matches `/etc/nut` and its original `2.8.1-5` package is usable for
   rollback. No new host scratch directory or credential copy was created.
4. Installed the tracked scoped APT source/pin. Simulation proposed exactly one
   upgrade (`nut-client`), no installations/removals or dependency upgrades.
   `libupsclient6t64` stayed at `2.8.1-5`.
5. Installed `2.8.1-5+deb13u1`, preserving conffiles. The package restarted NUT at
   10:09:01; TLS handshake completed at 10:09:02. No host or VM reboot occurred.
6. Confirmed enabled/active service and both running upsmon executable hashes
   match the official package. `/etc/nut` still matches the protected pre-update
   copy; secondary role, power value, TLS, real shutdown command and DEADTIME
   remain unchanged. Package verification reported only the two expected
   customized conffiles. No package is held.
7. Authenticated read-only UPS query after update returned `OL`, charge 100%,
   runtime 4,339 seconds. Both VMs remained running with successful guest-agent
   pings; no failed host units. `local`, `local-lvm` and `unas-pro` were active.
8. K3s was Ready; Argo `media`, `monitoring`, `sealed-secrets` were Synced/Healthy;
   all eight media Deployments were available and all nine PVCs bound.
9. Updated clean-install/maintenance/rollback documentation, scoped APT files,
   system policy and TODO; consolidated incident evidence and removed conflicting
   waiting/custom-build/simulator instructions. No public custom release is needed.
10. Executed the exact read-only status query in the runbook successfully;
    confirmed an established upsmon TCP connection, no subsequent NUT journal
    errors and `/run/nut` permissions matching the packaged tmpfiles rule.
    Reviewed NUT/UPS references across setup, infrastructure records and plans.
    Standards and spec reviews each identified the same two links to an
    untracked backup report; replaced them with committed recovery guidance.
    Corrected the touched TODO's stale monitoring link. Unrelated work remains
    outside this change.

No simulated UPS, self-test, FSD, mains removal or power-cycle test was performed,
following the owner's October 29 decision. Real failure behavior, playback,
Home Assistant functions and backup restoration were not exercised. Package/source
inspection and live monitoring checks establish deployment, not end-to-end outage
recovery. Lost communication while genuinely on battery can still legitimately
trigger conservative shutdown.

## Remaining work

- **APT source retirement:** when the fixed package enters normal stable sources,
  remove the scoped source/pin and reconcile repository configuration using the
  [maintenance procedure](../infrastructure/node-main/nut/README.md#routine-updates-and-source-retirement).
- **Communication loss:** cause remains unknown. Correlate host NUT/kernel logs
  with UPS-service and Study-switch port events at the dated incident times in
  [the evidence record](nut-incidents.md). If historical logs are unavailable,
  define bounded, redacted monitoring with retention for the next event; do not
  enable indefinite credential-bearing debug capture. Check live routes/ports
  and actual power cabling before blaming the unpowered gateway or an IP conflict.
- **Physical restart:** confirm BIOS AC-recovery/WOL behavior and UPS restore
  settings with the owner. A halted host may never see AC loss; guest onboot
  alone cannot restore it. Any power-cycle/outage test requires separate approval,
  backup readiness and owner presence.
- **Package warning:** the official unit still optionally references missing
  `nut-common-tmpfiles.conf`; it supplies `nut-client.conf`. Monitoring works.
  Track separately if it affects startup; no unit override was introduced.
- **Recovery assurance:** NAS storage is accessible, but the old intermittent
  denial's cause and backup restore readiness remain unresolved. Follow the
  [recovery guidance](../infrastructure/node-main/ns-argo/SETUP.md#recovery-checks); storage activity is not a
  restore test. Application-aware guest shutdown hardening is separate work.
