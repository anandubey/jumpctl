# Publishing `jumpcloud-agent-bin` to the AUR

> **Blocked as of 2026-09-06: new AUR account registration is disabled.**
> Everything below is ready to run the moment it reopens. Check
> <https://aur.archlinux.org/register> periodically — the AUR closes signups
> during spam waves and reopens without announcement. Until then, install
> straight from this repo with `makepkg -si` (see the README); the package is
> identical, only the distribution channel differs.

The package is verified and AUR-ready. Steps 1–2 are one-time account setup;
3–5 are the release itself.

## 1. AUR account

Register at <https://aur.archlinux.org/register> — separate from GitHub, and
currently disabled (see above). Confirm via email.

## 2. SSH key

The AUR accepts pushes over SSH only. Reuse an existing key or make a dedicated
one:

```sh
ssh-keygen -t ed25519 -C "aur" -f ~/.ssh/aur
cat ~/.ssh/aur.pub
```

Paste the public key into **My Account → SSH Public Key**, then point SSH at it:

```sh
cat >> ~/.ssh/config <<'EOF'

Host aur.archlinux.org
  User aur
  IdentityFile ~/.ssh/aur
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
```

Verify. A successful authentication prints `Interactive shell is disabled` —
that is the expected result, not an error:

```sh
ssh aur@aur.archlinux.org help
```

## 3. Clone the empty AUR repo

The repository name **must** equal `pkgname`. It does not exist server-side
yet; cloning creates it locally and warns about an empty repository, which is
normal:

```sh
git clone ssh://aur@aur.archlinux.org/jumpcloud-agent-bin.git ~/private/aur-jumpcloud-agent-bin
cd ~/private/aur-jumpcloud-agent-bin
```

## 4. Copy the build files

Ten files. Deliberately **not** `README.md`, `LICENSE`, `AUR_RELEASE.md` or
`.gitignore` — an AUR repo carries only what is needed to build:

```sh
cd ~/private/aur-jumpcloud-agent-bin
for f in PKGBUILD .SRCINFO jumpcloud-agent-bin.install jumpcloud-agent-ca.crt \
         jumpctl jumpctl-check.service jumpctl-check.timer \
         pam-password-auth pam-common-auth os-release-compat; do
  cp ~/private/jumpctl/$f .
done
git add -A && git status --short
```

## 5. Commit and push

```sh
git commit -m "feat(jumpctl): add jumpcloud-agent-bin 2.166.2"
git push -u origin master
```

Two things that trip people up:

- The AUR's default branch is **`master`**, not `main`.
- The server **rejects any push whose `.SRCINFO` disagrees with the
  `PKGBUILD`**. Regenerate it with `makepkg --printsrcinfo > .SRCINFO` and
  commit it in the same commit as any PKGBUILD change.

Then add a comment on the package page linking back to
<https://github.com/anandubey/jumpctl> so reviewers can find the reasoning
behind the OS compat shim.

## 6. Install

```sh
paru -S jumpcloud-agent-bin      # or: yay -S jumpcloud-agent-bin
sudo jumpctl compat on
sudo jumpctl enroll
```

The helper will prompt to import JumpCloud's signing key
`64A5EEB13230D7C20C364F191E6CC51448B05857`. That is the pin declared in
`validpgpkeys`; accepting it is expected.

## Releasing a new agent version

```sh
jumpctl check                                  # reports when GA moves ahead
# in the PKGBUILD: bump pkgver, reset pkgrel=1
updpkgsums                                     # refreshes sha256sums_x86_64
makepkg -f                                     # confirm it still builds
makepkg --printsrcinfo > .SRCINFO
git commit -am "feat(jumpctl): bump jcagent to <version>"
git push
```

Push to both this GitHub repo and the AUR repo; they are separate remotes.
Changing any local source file (`jumpctl`, the PAM shims, `os-release-compat`)
also requires refreshing its entry in `sha256sums=()`.

## Verification status

Confirmed in a CachyOS container before publishing:

| Check | Result |
|---|---|
| Clean build, nothing pre-staged | ✅ makepkg downloaded all 36 MB from the CDN |
| Source checksum | ✅ Passed |
| GPG signature against the pinned key | ✅ Passed |
| Install | ✅ no file conflicts, `pacman -Qkk` reports 0 altered files |
| Agent registration | ✅ completes, real system ID issued |
| Steady state | ✅ posts system reports, zero restarts |
| Removal | ✅ restores PAM/sshd backups, wipes `/opt/jc` |
| namcap on PKGBUILD | ✅ clean |

### Expected namcap findings on the built package

Both are inherent to repackaging a vendor binary. **Do not try to fix them.**

- `E: ELF files outside of a valid path ('opt/')` — JumpCloud installs to
  `/opt/jc` by design.
- `W: ELF file ... lacks FULL RELRO` — upstream's prebuilt binaries. They
  cannot be recompiled, and `options=('!strip')` is mandatory because the agent
  verifies `jumpcloudgo-chrome` against a shipped signature.

## One thing to be ready for

The AUR is a public, reviewed space, and this package ships `os-release-compat`
— a mechanism for reporting an untrue OS to a vendor. That is defensible: it is
off by default, the `.install` message states the tradeoff at install time, and
the README leads with a warning. But expect a comment about it eventually, and
answer plainly: JumpCloud's API rejects Arch at registration with HTTP 400, so
without it the agent cannot register at all and simply crash-loops.
