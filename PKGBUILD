# Maintainer: Eduardo Lúcio Amorim Costa <mardoqueu.acess@gmail.com>
#
# GlobalProtect VPN Client (Palo Alto Networks) packaging for Arch Linux
# and Arch-based distros (Manjaro, EndeavourOS, etc.). Based on:
#   - AUR globalprotect-bin (Timothy Gelter)
#   - github.com/Nukesor/globalprotect-pkg
#
# Single source: the official PanGPLinux-<ver>.tgz archive from Palo Alto.
# We extract only the UI RPM from it (which already bundles CLI + daemon).
# The download URL below is a third-party mirror — if it ever breaks, see
# README.md (section "Download failure") for how to obtain the archive
# manually and drop it next to this PKGBUILD.

pkgname=globalprotect
pkgver=6.3.3.1
pkgrel=619
pkgdesc="GlobalProtect VPN client Agent (Palo Alto Networks)"
arch=('x86_64')
url="https://docs.paloaltonetworks.com/globalprotect/6-3/globalprotect-app-user-guide/globalprotect-app-for-linux"
license=('custom')
depends=('qt5-webkit' 'wmctrl')
makedepends=('libarchive')
provides=('globalprotect-bin')
conflicts=('globalprotect-bin')
# Binaries and .so files come pre-built from Palo Alto — skip strip/debug.
options=(!strip !debug staticlibs)
install=globalprotect.install

_upstream_tgz="PanGPLinux-6.3.3-c22.tgz"
_upstream_url="https://isssweb.umkc.edu/files/${_upstream_tgz}"
_ui_rpm="GlobalProtect_UI_rpm-${pkgver}-${pkgrel}.rpm"

source=("${_upstream_url}")
# The upstream archive ships multiple variants (deb/rpm, x86/arm). We don't
# want makepkg auto-extracting all of them — we handle it manually.
noextract=("${_upstream_tgz}")
sha256sums=('e23ab15b813aaae577f19bfd61ef524b39d4bd5f55dd56c7db3076044978f04e')

_folder="/opt/paloaltonetworks/globalprotect"

prepare() {
    cd "${srcdir}"

    tar -xzf "${_upstream_tgz}" "./${_ui_rpm}"
    bsdtar -xf "${_ui_rpm}"
    rm -f "${_ui_rpm}"

    # PanGPS validates the distro via /etc/os-release and bails out on
    # anything other than Ubuntu. The "Ubuntu" string is hardcoded in the
    # binary — replacing it with another 6-byte string is enough to make
    # the check pass. Trick inherited from Nukesor's package.
    local search=Ubuntu
    local replace
    replace=$(echo "Arch L" | head -c${#search})
    sed -i "s/${search}/${replace}/g" "${srcdir}${_folder}/PanGPS"
}

package() {
    cd "${srcdir}"

    cp -a opt "${pkgdir}/opt"
    cp -a etc "${pkgdir}/etc"
    cp -a usr "${pkgdir}/usr"

    install -Dm644 "${srcdir}${_folder}/gpd.service" \
        "${pkgdir}/usr/lib/systemd/system/gpd.service"

    # The upstream RPM only ships an xdg autostart file for the UI
    # (PanGPUI). 6.3.3 also ships a gpa.service, but only as a static
    # file in /opt/ — it isn't activated anywhere. Without the user-space
    # PanGPA agent running first, both UI and CLI hang with "Cannot
    # connect to local gpd service". Existing PKGBUILDs work around this
    # via /etc/profile.d (AUR) or a gpa.service user unit (Nukesor). Here
    # we use xdg-autostart — symmetric to the autostart already present
    # for PanGPUI, works on any DE under X11 or Wayland.
    install -d "${pkgdir}/etc/xdg/autostart"
    cat > "${pkgdir}/etc/xdg/autostart/PanGPA.desktop" <<'EOF'
[Desktop Entry]
Type=Application
Name=PanGPA
Exec=/opt/paloaltonetworks/globalprotect/PanGPA start
Terminal=false
NoDisplay=true
X-GNOME-Autostart-Phase=Initialization
EOF
    chmod 644 "${pkgdir}/etc/xdg/autostart/PanGPA.desktop"

    install -d "${pkgdir}/usr/bin"
    ln -sf "${_folder}/globalprotect" "${pkgdir}/usr/bin/globalprotect"
    ln -sf "${_folder}/PanGPUI" "${pkgdir}/usr/bin/PanGPUI"
}
