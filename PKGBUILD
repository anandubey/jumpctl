# Maintainer: Anand Dubey <dubey.anandkr@gmail.com>

# Repackages JumpCloud's official signed RPM. Upstream ships no Arch package and
# its installer cannot be used here, for two independent reasons:
#
#   1. kickstart.jumpcloud.com/Detect is sent "$NAME $VERSION" from os-release
#      and rejects anything not on JumpCloud's supported list. On Arch/CachyOS
#      the kickstart aborts there -- before it writes /opt/jc/ca.crt -- so the
#      CA bundle is shipped by this package instead.
#   2. The RPM's own scriptlets `case` on $ID from os-release; `arch`/`cachyos`
#      fall through to the catch-all, log "unsupported os" and exit 0. The PAM
#      modules therefore never reach the security dir and the unit is never
#      enabled. That work is done in package() and the .install file.

pkgname=jumpcloud-agent-bin
pkgver=2.166.2
pkgrel=1
pkgdesc='JumpCloud device management agent (repackaged upstream binary)'
arch=('x86_64')
url='https://github.com/anandubey/jumpctl'
license=('LicenseRef-JumpCloud-Proprietary')

# Two sources, because linkage alone is not enough.
#
# Library deps -- what `ldd` on /opt/jc/lib/*.so resolves (the Go binaries are
# all statically linked, so only the PAM modules contribute here):
#   libpam.so.0 -> pam            libaudit.so.1  -> audit
#   libcap-ng.so.0 -> libcap-ng   libsystemd.so.0 -> systemd-libs
#   libgcc_s.so.1 -> gcc-libs     libc/libdl/libpthread/libresolv -> glibc
#
# Command deps -- the agent embeds a YAML dependency manifest declaring what it
# shells out to. These are the rpm-side entries translated to Arch names, kept
# only where the manifest's own "Code:" annotation shows real agent code calling
# them: nss(certutil) for certificates/nssdb/nssdb_manager.go, dmidecode for
# systemreport/.../data_collectors.go, psmisc/shadow/curl for the install and
# user-management scripts. Entries the manifest marks "not directly invoked by
# agent code" (lsof, net-tools) are optdepends instead.
depends=('glibc' 'pam' 'audit' 'libcap-ng' 'systemd-libs' 'gcc-libs'
         'sudo' 'curl' 'nss' 'dmidecode' 'psmisc' 'shadow')
optdepends=('openssh: JumpCloud SSH key and MFA management'
            'lsb-release: richer OS reporting to the Admin Console'
            'lsof: manual diagnostics (agent_diagnostics.sh)'
            'net-tools: manual network diagnostics'
            'chromium: JumpCloud Go browser integration'
            'firefox: JumpCloud Go browser integration')
provides=("jcagent=${pkgver}")
conflicts=('jcagent')
install="${pkgname}.install"

# The agent rewrites its PAM stack in place once console policy calls for it,
# so these must survive reinstalls and upgrades untouched.
backup=('etc/pam.d/password-auth' 'etc/pam.d/common-auth')

# !strip is mandatory: the agent verifies /opt/jc_user_ro/bin/jumpcloudgo-chrome
# against the shipped jumpcloudgo-chrome.sig using /opt/jc/trustedkeys.gpg.
# Stripping the binary invalidates that signature.
options=('!strip' '!debug' '!lto' 'emptydirs')

# JumpCloud (JumpCloud package signing key) <support@jumpcloud.com>
#   gpg --import <(curl -s https://cdn02.jumpcloud.com/production/versions/2.166.2/trustedKeys.gpg)
#
# This pin is the only real integrity check on the payload: on a network doing
# TLS inspection the HTTPS chain proves nothing about origin.
validpgpkeys=('64A5EEB13230D7C20C364F191E6CC51448B05857')

_cdn="https://cdn02.jumpcloud.com/production/versions/${pkgver}"
source=("${pkgname}.install"
        'jumpcloud-agent-ca.crt'
        'jumpctl'
        'jumpctl-check.service'
        'jumpctl-check.timer'
        'pam-password-auth'
        'pam-common-auth'
        'os-release-compat')
# JumpCloud also publishes jcagent-linux-rpm-aarch64.rpm. It is not declared
# here because it has never been built or run on this package's behalf; add
# aarch64 to arch() with its own sums once someone verifies it on ARM.
source_x86_64=("${pkgname}-${pkgver}-x86_64.rpm::${_cdn}/jcagent-linux-rpm-x86_64.rpm"
               "${pkgname}-${pkgver}-x86_64.rpm.sig::${_cdn}/jcagent-linux-rpm-x86_64.rpm.sig")

sha256sums=('SKIP' 'SKIP' 'SKIP' 'SKIP' 'SKIP' 'SKIP' 'SKIP' 'SKIP')
sha256sums_x86_64=('73318daf1d4dc524ed03fd90639873a80bb12cd42efb4e9489d1b8a314ed009c'
                   'SKIP')

package() {
  # libarchive reads the RPM header as a filter and unpacks the cpio payload.
  bsdtar -xf "${srcdir}/${pkgname}-${pkgver}-${CARCH}.rpm" -C "${pkgdir}"

  # Arch merges /lib -> usr/lib and /usr/lib64 -> lib. Extracting upstream's
  # layout verbatim creates real directories where Arch has symlinks, and
  # pacman aborts the install on that conflict.
  install -dm755 "${pkgdir}/usr/lib/systemd"
  cp -a "${pkgdir}/lib/systemd/." "${pkgdir}/usr/lib/systemd/"
  rm -rf "${pkgdir}/lib"

  install -dm755 "${pkgdir}/usr/lib/mozilla/native-messaging-hosts"
  cp -a "${pkgdir}/usr/lib64/mozilla/." "${pkgdir}/usr/lib/mozilla/"
  rm -rf "${pkgdir}/usr/lib64"

  # Upstream's posttrans copies these out of /opt/jc/lib into a per-distro
  # libDir at install time, and skips it entirely on Arch. Ship them directly
  # so pacman owns them and removes them cleanly.
  install -dm755 "${pkgdir}/usr/lib/security"
  install -m644 "${pkgdir}/opt/jc/lib/"*.so "${pkgdir}/usr/lib/security/"

  # The agent knows only RHEL (password-auth) and Debian (common-auth) stack
  # names; Arch has neither. Without these, an agent-written PAM line would
  # reference a missing file and break authentication for that service.
  install -Dm644 "${srcdir}/pam-password-auth" "${pkgdir}/etc/pam.d/password-auth"
  install -Dm644 "${srcdir}/pam-common-auth" "${pkgdir}/etc/pam.d/common-auth"

  # The hourly updater runs /opt/jc/jcUpdate.sh, which drives yum/dnf/apt to
  # reinstall the agent. It cannot work under pacman, so it is not shipped
  # rather than left to fail on a timer. `jumpctl check` replaces it.
  rm -f "${pkgdir}/etc/cron.d/jumpcloud-updater"
  rmdir --ignore-fail-on-non-empty "${pkgdir}/etc/cron.d"

  # kickstart.sh normally drops this in, but it aborts on Arch before reaching
  # that point. The agent needs it to validate the enrolment endpoint.
  install -Dm600 "${srcdir}/jumpcloud-agent-ca.crt" "${pkgdir}/opt/jc/ca.crt"

  install -Dm755 "${srcdir}/jumpctl" "${pkgdir}/usr/bin/jumpctl"

  # Data file only -- inert until `jumpctl compat on` writes the systemd
  # drop-in that binds it into the agent's mount namespace. Installing it does
  # not change how this host identifies itself.
  install -Dm644 "${srcdir}/os-release-compat" \
    "${pkgdir}/usr/share/jumpcloud-agent/os-release-compat"
  # Ships disabled; Arch does not auto-enable units.
  install -Dm644 "${srcdir}/jumpctl-check.service" "${pkgdir}/usr/lib/systemd/system/jumpctl-check.service"
  install -Dm644 "${srcdir}/jumpctl-check.timer" "${pkgdir}/usr/lib/systemd/system/jumpctl-check.timer"

  chmod 700 "${pkgdir}/opt/jc"
}
