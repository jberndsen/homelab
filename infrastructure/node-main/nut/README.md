# NUT on Proxmox: package maintenance and recovery

Use official Debian packages with the CAL-clearing fix. The reviewed trixie
amd64 package is `nut-client 2.8.1-5+deb13u1`. It clears an obsolete calibration
flag when UPS status no longer contains `CAL`; low-battery/FSD handling remains
unchanged. This addresses both false shutdown after communication loss with
stale CAL and suppressed low-battery shutdown after calibration.

Debian accepted this stable-targeted update into **trixie-proposed-updates** on
2026-11-04. This is the staging suite for a future stable point release, not
trixie-updates or security. The deployed APT policy admits only this exact
`nut-client` version from that suite. All other proposed updates are excluded.
No custom build, GitHub binary release, package hold or unattended updates are
needed. Newer versions from normal stable/security repositories can supersede
it through ordinary reviewed APT maintenance.

Sources: [Debian bug and upload](https://bugs.debian.org/1138630),
[upstream fix](https://github.com/networkupstools/nut/pull/2169),
[Debian proposed-updates policy](https://www.debian.org/releases/proposed-updates).
Deployment and validation evidence lives in [the execution record](../../../plans/nut-update.md).

## Clean installation or update

The procedure targets Debian 13/trixie amd64 on Proxmox. Verify the release and
architecture first. Do not add testing/sid or upgrade the distribution for NUT.
Networking, credentials, monitor role and VM startup configuration are covered
by [infrastructure recovery](../../../setup.MD#ups-shutdown-on-proxmox).

On Proxmox as root:

```sh
cat /etc/os-release
dpkg --print-architecture
apt-get update
apt-cache policy nut-client
apt-mark showhold
```

If a fixed candidate is already available from regular trixie/security/updates,
use it directly after reviewing its source changes. If the candidate is still
`2.8.1-5`, use the scoped source below. If the expected package is unavailable,
stop and reassess official availability; never silently install the unfixed
version or disable shutdown protection.

From this repository on the laptop, install the pin **before** enabling the
source (SSH uses the owner's normal authentication):

```sh
ssh root@192.168.30.5 'install -m 0644 /dev/stdin /etc/apt/preferences.d/nut-stable-fix.pref' < infrastructure/node-main/nut/nut-stable-fix.pref
ssh root@192.168.30.5 'install -m 0644 /dev/stdin /etc/apt/sources.list.d/nut-stable-fix.sources' < infrastructure/node-main/nut/nut-stable-fix.sources
```

The files use Debian's archive signing key. Do not use `trusted=yes`, disable
signature checking or add a general proposed-updates target release.
On Proxmox:

```sh
apt-get update
apt-cache policy nut-client libupsclient6t64
apt-get -s --no-install-recommends install nut-client
```

For the scoped package, `nut-client` should be candidate `2.8.1-5+deb13u1` at
priority 500; other proposed packages should be priority -1. The update from
`2.8.1-5` requires only `nut-client`; its existing client library is compatible.
A fresh install can also need normal stable dependencies. Review the simulation
and stop on unrelated upgrades/removals or dependency changes.

Before changing an existing monitor, confirm UPS status `OL`, adequate battery,
working host access and a protected configuration/previous-package recovery
copy. Keep new scratch/download/configuration-backup files under this repo's
ignored `tmp/` with restrictive permissions, including during transfers. Do not
print credentials. The package restarts NUT, briefly interrupting monitoring;
keep the host and VMs running. For a fresh installation, restore credentials
from the password manager and finish the linked setup before relying on NUT.

```sh
apt-get --no-install-recommends -o Dpkg::Options::=--force-confold install nut-client
dpkg-query -W nut-client libupsclient6t64
dpkg --compare-versions "$(dpkg-query -W -f='${Version}' nut-client)" ge '2.8.1-5+deb13u1'
systemctl is-enabled nut-monitor
systemctl is-active nut-monitor
journalctl -u nut-monitor --since '5 minutes ago' --no-pager
```

Verify the version comparison succeeds. Preserve `MODE=netclient`, secondary
role, power value, TLS, credentials and shutdown command. The existing TLS
policy does not validate the UPS certificate; changing that is separate work.
For a fresh install, follow the linked setup's enable/start step after editing
configuration. Never introduce a hold for this official fix.

## Verify monitoring

Check the journal for a completed TLS handshake and no authentication failures,
repeated poll errors or shutdown requests. Inspect both upsmon processes:

```sh
systemctl show nut-monitor -p MainPID -p ActiveState -p SubState
for pid in $(pgrep -x upsmon); do
    readlink /proc/$pid/exe
    sha256sum /proc/$pid/exe
done
sha256sum /usr/lib/nut/upsmon
dpkg -V nut-client
qm guest cmd 102 ping
qm guest cmd 103 ping
qm list
systemctl --failed --no-pager
```

Process executables must match the installed binary, with no `(deleted)` path.
`dpkg -V` may report the deliberately customized NUT conffiles; compare those
privately to the pre-update copy. Do not publish their contents. Check K3s with
the laptop's `kubectl get nodes -o wide`, then Argo applications and media
Deployments. Full media playback and backup restoration are separate tests.

The following read-only query runs on Proxmox, reads credentials in memory and
prints only status/charge/runtime. It uses the existing unverified TLS policy.
An anonymous `upsc` request is denied by this UPS. Do not put passwords in shell
arguments, history or documentation.

```sh
python3 - <<'PY'
import shlex
import socket
import ssl
from pathlib import Path

rows = [shlex.split(line) for line in
        Path('/etc/nut/upsmon.conf').read_text().splitlines()
        if line.strip() and not line.lstrip().startswith('#')]
monitor = next(row for row in rows if row[0] == 'MONITOR')

def receive(sock):
    data = b''
    while not data.endswith(b'\n'):
        part = sock.recv(1)
        if not part:
            raise RuntimeError('UPS disconnected')
        data += part
    return data.decode().strip()

def quoted(value):
    return '"' + value.replace('\\', '\\\\').replace('"', '\\"') + '"'

sock = socket.create_connection(('192.168.1.34', 3493), timeout=5)
sock.sendall(b'STARTTLS\n')
assert receive(sock) == 'OK STARTTLS', 'TLS unavailable'
context = ssl._create_unverified_context()
with context.wrap_socket(sock, server_hostname='192.168.1.34') as secure:
    for command, value in [('USERNAME', monitor[3]), ('PASSWORD', monitor[4])]:
        secure.sendall((command + ' ' + quoted(value) + '\n').encode())
        assert receive(secure) == 'OK', 'Authentication failed'
    for key in ['ups.status', 'battery.charge', 'battery.runtime']:
        secure.sendall(('GET VAR ups ' + key + '\n').encode())
        response = receive(secure)
        assert response.startswith('VAR ups ' + key + ' '), 'Status query failed'
        print(response)
PY
```

The Debian patch does **not** add upstream's `NOTCAL` notification or
“calibration finished” string. Their absence does not mean this patch is missing.
Verify the actual source patch and package/binary identity instead. The vendor
unit still references absent `nut-common-tmpfiles.conf` in an optional
`ExecStartPre`; the package supplies `nut-client.conf`, and systemd creates the
runtime directory through that tmpfiles entry. A warning alone is not evidence
that monitoring failed. Check `/run/nut`, service state and connectivity; do not
create a fake common package or edit the vendor unit to hide the warning.

No simulated or physical failure testing is part of this procedure, per owner
decision. A future outage test requires a separate maintenance window, backup
readiness and a recovery method. Do not invoke FSD, trigger a self-test, remove
mains or cycle UPS output as an installation check.

## Routine updates and source retirement

Run ordinary reviewed APT updates and verify NUT after a package restart. The
exact-version pin is an exception allowing that release, not a hold or ceiling:
newer stable/security candidates remain eligible. New proposed versions are
excluded until reviewed. Recheck Debian bug/source provenance when selecting a
replacement; do not rely on version strings from an unrelated distribution.

Once `apt-cache policy nut-client` shows the fixed version or a reviewed newer
one in regular trixie, trixie-updates or trixie-security:

1. Remove only `/etc/apt/sources.list.d/nut-stable-fix.sources` and
   `/etc/apt/preferences.d/nut-stable-fix.pref` from the host.
2. Refresh APT and verify the fixed candidate still comes from regular sources.
3. Simulate/install any newer version and repeat the monitoring checks.
4. Remove the two retired configuration files from this directory and update
   this procedure, setup links, system record and TODO together.

Do not remove the scoped source before a clean install can obtain the fixed
package from regular repositories. Do not pin all proposed updates at 500.

## Rollback

Rollback to `2.8.1-5` restores the stale-CAL defect. Use it only to recover failed
monitoring while investigating a regression; never leave NUT stopped. Preserve
configuration, inspect the simulation and verify monitoring afterwards:

```sh
apt-get -s --no-install-recommends install nut-client=2.8.1-5
apt-get --allow-downgrades --no-install-recommends -o Dpkg::Options::=--force-confold install nut-client=2.8.1-5
systemctl restart nut-monitor
systemctl is-active nut-monitor
journalctl -u nut-monitor --since '5 minutes ago' --no-pager
```

If that version has left the repositories, use the protected original `.deb`
retained before maintenance (`apt-get install /absolute/path/to/package.deb`,
with `--allow-downgrades` and configuration preservation as above). The
[execution record](../../../plans/nut-update.md) identifies the protected recovery
assets verified for that deployment. These are not prerequisites for a clean
installation. Never publish configuration backups.
Re-run live UPS/guest checks and explicitly track the restored defect; the
normal candidate remains the fixed version, so resolve the regression before
subsequent routine updates.
