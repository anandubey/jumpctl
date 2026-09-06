# jumpctl

Arch Linux packaging for the **JumpCloud device management agent**, plus a small
CLI to enrol and maintain it. JumpCloud ships no Arch package and its installer
refuses to run here; this repo produces `jumpcloud-agent-bin`, an AUR package
built from JumpCloud's official GPG-signed RPM.

Built and tested on CachyOS with systemd. Agent version **2.166.2**.

## Install

```sh
gpg --import <(curl -s https://cdn02.jumpcloud.com/production/versions/2.166.2/trustedKeys.gpg)
git clone https://github.com/anandubey/jumpctl && cd jumpctl
makepkg -si
sudo jumpctl enroll
```

`jumpctl enroll` prompts for your connect key (Admin Console → Devices → **+** →
Linux) rather than taking it as an argument, so it stays out of shell history
and the process list.

```
jumpctl enroll    Enrol this device (needs root)
jumpctl check     Report whether the agent is behind upstream GA
jumpctl status    Show package, enrolment and service state
```

## Why upstream's installer doesn't work

JumpCloud's kickstart fails on Arch at two independent points:

1. **Server-side.** Stage 1 POSTs `arch=$(uname -m)&os=$NAME $VERSION` to
   `/Detect` and aborts if JumpCloud doesn't recognise the OS. On CachyOS it
   dies there — *before* it writes `/opt/jc/ca.crt`, so that CA bundle is
   shipped by this package instead.
2. **Client-side.** The RPM's scriptlets `case` on `$ID` from `/etc/os-release`.
   `arch`/`cachyos` fall through to the catch-all, log `unsupported os` and
   `exit 0` — so the PAM modules are never installed and the unit never
   enabled. `package()` and the `.install` file do that work.

Enrolment itself needs no installer: given `ca.crt` and `agentBootstrap.json`,
the agent generates a CSR and exchanges it at `kickstart.jumpcloud.com/SignCsr`
on first start.

## What this package changes vs. the upstream RPM

| Upstream | Here | Why |
|---|---|---|
| `/lib/systemd/…` | `/usr/lib/systemd/…` | `/lib` is a symlink on Arch; pacman refuses the conflict |
| `/usr/lib64/mozilla/…` | merged into `/usr/lib/mozilla/…` | `/usr/lib64` is a symlink on Arch |
| PAM `.so` copied at install time | shipped in `/usr/lib/security/` | pacman owns and removes them cleanly |
| `/etc/cron.d/jumpcloud-updater` | **not shipped** | `jcUpdate.sh` drives `yum`/`dnf`/`apt` |
| `ca.crt` written by kickstart | shipped in package | kickstart aborts before writing it |
| — | `/etc/pam.d/{password-auth,common-auth}` | compatibility shims, see below |

`options=('!strip')` is **mandatory** — the agent verifies `jumpcloudgo-chrome`
against the shipped `.sig` using `/opt/jc/trustedkeys.gpg`, and stripping
invalidates that signature.

## The PAM shims

This is the sharpest edge in the port. The agent knows exactly two PAM stack
layouts:

- `password-auth` + `system-auth` → RHEL/Fedora
- `common-auth` → Debian/Ubuntu

**Arch has neither `password-auth` nor `common-auth`.** If console policy makes
the agent write `auth substack password-auth` into `/etc/pam.d/sshd`, PAM would
reference a file that does not exist and authentication for that service breaks
— a real lockout path.

Both names are therefore installed as thin shims that `include system-auth`,
which carries the same four management groups as Fedora's `password-auth`. They
are listed in `backup=()` so the agent's own edits survive upgrades.

If you enable MFA or password policies in the Admin Console, **keep a root shell
open and test authentication in a second session** before logging out.

## Updating

There is no auto-updater, by design — `jcUpdate.sh` is delivered by JumpCloud at
update time and drives `yum`/`dnf`/`apt`, so porting it would mean falsifying
`/etc/os-release` system-wide and shimming a root-run, server-controlled script.
Instead:

```sh
sudo systemctl enable --now jumpctl-check.timer   # daily, reports only
```

`jumpctl check` reads the version from the RPM header of JumpCloud's
unauthenticated GA pointer via a 16 KB range request, rather than downloading
36 MB. It never installs anything. To upgrade: bump `pkgver`, refresh
`sha256sums_x86_64`, rebuild.

## Removal

`pacman -R jumpcloud-agent-bin` stops the service, restores the `.orig` backups
of `sshd_config`/`pam.d`/`login.defs`, and deletes `/opt/jc` — **including the
enrolled client keypair**. Also delete the device from the Admin Console;
removing it locally does not deregister it.

## Verification

The payload is checked against JumpCloud's signing key, pinned in the PKGBUILD:

```
64A5EEB13230D7C20C364F191E6CC51448B05857
JumpCloud (JumpCloud package signing key) <support@jumpcloud.com>
```

This matters more than usual: on a network doing TLS inspection (corporate
proxy, Cloudflare Gateway) the HTTPS chain proves nothing about origin, so the
detached signature is the only real integrity check on the binary.

## Status

Verified in a CachyOS container: builds, signature verifies, installs with no
file conflicts, all PAM modules resolve, the unit enables and starts, the agent
runs and reaches the enrolment endpoint, and removal cleans up fully.

**Not yet verified:** enrolment with a live connect key (needs a real key),
user/sudoers provisioning, and the `aarch64` build.

## Licence

Packaging and `jumpctl` are MIT (see `LICENSE`). The JumpCloud agent itself is
proprietary software owned by JumpCloud, Inc., downloaded at build time from
JumpCloud's CDN and redistributed by nobody — this repo contains no JumpCloud
binaries.
