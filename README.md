# jumpctl

Arch Linux packaging for the **JumpCloud device management agent**, plus a small
CLI to enrol and maintain it. JumpCloud ships no Arch package, its installer
refuses to run here, and its registration API rejects Arch outright. This repo
produces `jumpcloud-agent-bin`, an AUR package built from JumpCloud's official
GPG-signed RPM, and works around all three.

Verified end to end on CachyOS: the agent registers, reports, and holds steady
state. Agent version **2.166.2**.

> **Read [The OS gate](#the-os-gate) before installing.** Registering on Arch
> requires telling JumpCloud that this host is Fedora. That is a decision with
> consequences for your vendor relationship, and it is off by default.

Not yet on the AUR — new account registration is disabled there as of
2026-09-06. Install from this repo with `makepkg -si`; see
[AUR_RELEASE.md](AUR_RELEASE.md) for the publishing steps, ready to run when
signups reopen.

## Install

```sh
gpg --import <(curl -s https://cdn02.jumpcloud.com/production/versions/2.166.2/trustedKeys.gpg)
git clone https://github.com/anandubey/jumpctl && cd jumpctl
makepkg -si

sudo jumpctl compat on     # required to register -- see below
sudo jumpctl enroll
```

`jumpctl enroll` prompts for your connect key (Admin Console → Devices → **+** →
Linux) rather than taking it as an argument, so it stays out of shell history
and the process list.

```
jumpctl enroll         Enrol this device (needs root)
jumpctl check          Report whether the agent is behind upstream GA
jumpctl status         Show package, enrolment, service and compat state
jumpctl compat on|off  Toggle the os-release compatibility shim
```

## The OS gate

JumpCloud's registration API builds an OS string from `"$NAME $VERSION"` in
`/etc/os-release`. CachyOS ships no `VERSION` field, so the agent sends
`"CachyOS Linux "` and the server answers:

```
Could not get the Linux template ID from the server for arch='x86_64', os='CachyOS Linux '
Template ID request failed with status=400
```

No template ID means registration can never complete: the agent signs a client
certificate, fails, deletes it and retries forever. The same allowlist rejects
Arch independently in the kickstart installer's `/Detect` call, so this is a
deliberate server-side policy, not a parsing accident.

`jumpctl compat on` writes a systemd drop-in giving **the agent alone** a
substitute os-release identifying as Fedora 42:

```ini
[Service]
BindReadOnlyPaths=/usr/share/jumpcloud-agent/os-release-compat:/etc/os-release
BindReadOnlyPaths=/usr/share/jumpcloud-agent/os-release-compat:/usr/lib/os-release
```

Because this lives in the service's private mount namespace, `/etc/os-release`
is never modified and every other process still sees the truth:

```
$ jumpctl status
os compat:   on (Fedora Linux 42 (Workstation Edition))
$ grep ^NAME= /etc/os-release
NAME="CachyOS Linux"
```

There is deliberately **no revert-after-registration step**. The agent does
re-register on its own — it deletes its certificates and restarts the flow on
certain errors — so a shim that disappeared after first registration would
brick the device later, with no obvious cause.

Fedora 42 is chosen because the agent's own embedded dependency manifest lists
it as supported, and because this package installs JumpCloud's rpm payload, so
the rpm-side code paths the agent then selects match what is actually on disk.

### What it costs you

- **Vendor support will decline to help.** You are reporting an untrue OS.
- **Console-pushed software actions will fail.** With `ID=fedora` the agent
  targets `dnf`, which does not exist here.
- **One cosmetic error per report cycle:** `rpm -qi nss → exit status 127`,
  because the agent probes an RPM database Arch has not got. Costs one field in
  system insights.
- Possibly a terms-of-service question. That is your call to make.

Device management, users, sudoers and PAM are unaffected and work normally.

## The PAM shims

The sharpest edge in the port, and it is not theoretical — it fired during
testing. The agent knows exactly two PAM stack layouts:

- `password-auth` + `system-auth` → RHEL/Fedora
- `common-auth` → Debian/Ubuntu

**Arch has neither `password-auth` nor `common-auth`.** With compat on, the
agent believes it is Fedora and duly wrote this into `/etc/pam.d/sshd`:

```
auth substack password-auth
```

On stock Arch that references a file which does not exist, and SSH
authentication breaks — a real lockout. Both names therefore ship as thin shims
that `include system-auth`, which carries the same four management groups as
Fedora's `password-auth`. They are listed in `backup=()` so the agent's own
edits survive upgrades.

If you enable MFA or password policies, **keep a root shell open and test
authentication in a second session** before logging out.

## TLS inspection breaks enrolment

The agent validates `private-kickstart.jumpcloud.com` against its *pinned*
`/opt/jc/ca.crt`, not the system trust store. On a network doing TLS
interception — Cloudflare WARP/Gateway, or a corporate proxy — the presented
certificate has nothing to chain to and enrolment fails:

```
CheckAccess failed ... x509: certificate signed by unknown authority
```

Disconnect the inspecting proxy while enrolling. Adding the interception CA to
`/opt/jc/ca.crt` would also work but defeats the pinning, so it is not
recommended.

## What this package changes vs. the upstream RPM

| Upstream | Here | Why |
|---|---|---|
| `/lib/systemd/…` | `/usr/lib/systemd/…` | `/lib` is a symlink on Arch; pacman refuses the conflict |
| `/usr/lib64/mozilla/…` | merged into `/usr/lib/mozilla/…` | `/usr/lib64` is a symlink on Arch |
| PAM `.so` copied at install time | shipped in `/usr/lib/security/` | pacman owns and removes them cleanly |
| `/etc/cron.d/jumpcloud-updater` | **not shipped** | `jcUpdate.sh` drives `yum`/`dnf`/`apt` |
| `ca.crt` written by kickstart | shipped in package | kickstart aborts before writing it |
| — | `/etc/pam.d/{password-auth,common-auth}` | compatibility shims |
| — | `os-release-compat` | inert until `jumpctl compat on` |

`options=('!strip')` is **mandatory** — the agent verifies `jumpcloudgo-chrome`
against the shipped `.sig` using `/opt/jc/trustedkeys.gpg`, and stripping
invalidates that signature.

## Updating

No auto-updater, by design — `jcUpdate.sh` is delivered by JumpCloud at update
time and drives `yum`/`dnf`/`apt`. Instead:

```sh
sudo systemctl enable --now jumpctl-check.timer   # daily, reports only
```

`jumpctl check` reads the version from the RPM header of JumpCloud's
unauthenticated GA pointer via a 16 KB range request rather than downloading
36 MB, and never installs anything. To upgrade: bump `pkgver`, refresh
`sha256sums_x86_64`, rebuild.

## Removal

`pacman -R jumpcloud-agent-bin` stops the service, restores the `.orig` backups
of `sshd_config`/`pam.d`/`login.defs`, and deletes `/opt/jc` — **including the
enrolled client keypair**. Run `jumpctl compat off` first if you want the
drop-in gone too; it lives in `/etc` and pacman does not own it.

Also delete the device from the Admin Console. Removing it locally does not
deregister it.

## Verification

The payload is checked against JumpCloud's signing key, pinned in the PKGBUILD:

```
64A5EEB13230D7C20C364F191E6CC51448B05857
JumpCloud (JumpCloud package signing key) <support@jumpcloud.com>
```

This matters more than usual: on a network doing TLS inspection the HTTPS chain
proves nothing about origin, so the detached signature is the only real
integrity check on the binary.

## Status

Verified on CachyOS: builds, signature verifies, installs with no file
conflicts, all PAM modules resolve, unit enables and starts, the agent enrols,
**completes registration**, posts system reports and holds steady state with
zero restarts. Removal cleans up fully.

**x86_64 only.** JumpCloud does publish an ARM64 build, but this package does
not declare `aarch64` because it has never been built or run on ARM. Add it once
someone verifies it on real hardware.

## Licence

Packaging and `jumpctl` are MIT (see `LICENSE`). The JumpCloud agent itself is
proprietary software owned by JumpCloud, Inc., downloaded at build time from
JumpCloud's CDN — this repo contains no JumpCloud binaries.
