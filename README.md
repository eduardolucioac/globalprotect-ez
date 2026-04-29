# globalprotect-ez

![globalprotect-ez](./images/globalprotect-ez.png)

Arch Linux package for the Palo Alto Networks **GlobalProtect** VPN client
(version **6.3.3.1-619**), built from the official `PanGPLinux-6.3.3-c22.tgz`
archive. Works on Arch Linux and any Arch-based distro (Manjaro,
EndeavourOS, CachyOS, etc.).

**IMPORTANT:** My life, my work and my passion is free software. Corrections, tweaks and improvements are very welcome (**pull requests** 😉)! Please consider giving us a ⭐, fork, support this project or even visit our professional profile (see [About](#about)). **Thanks!** 🤗

**Support free software and my work!** ❤️🐧

## Table of Contents

- [Layout](#layout)
- [Installation](#installation)
   * [1. Prerequisites](#1-prerequisites)
   * [2. Grab this project](#2-grab-this-project)
   * [3. Build and install the package](#3-build-and-install-the-package)
   * [4. Start the services](#4-start-the-services)
   * [5. Open the interface](#5-open-the-interface)
- [Useful commands](#useful-commands)
- [Optional configuration](#optional-configuration)
   * [Corporate proxy](#corporate-proxy)
   * [Client certificate (mTLS)](#client-certificate-mtls)
- [Harmless log messages](#harmless-log-messages)
   * [Info: missing `gp_excluded_users.txt`](#info-missing-gp_excluded_userstxt)
- [Upgrading an existing install](#upgrading-an-existing-install)
   * [Reinstalling the same version](#reinstalling-the-same-version)
- [Download failure](#download-failure)
- [Uninstall](#uninstall)
- [Upgrading to a newer upstream version](#upgrading-to-a-newer-upstream-version)
- [About](#about)

## Layout

```
globalprotect-ez/
├── PKGBUILD                   # build recipe
├── globalprotect.install      # post_install / post_remove messages
├── .SRCINFO                   # AUR metadata (optional here)
├── .gitignore
└── README.md                  # this file
```

The `PanGPLinux-6.3.3-c22.tgz` archive is **not** versioned with this
project — `makepkg` downloads it automatically from the URL declared in
the `PKGBUILD`.

---

## Installation

### 1. Prerequisites

Runtime and build dependencies:

```sh
yay -S --needed base-devel wmctrl
```

The package requires `qt5-webkit`, which is **no longer in the official
Arch repositories** — it has to come from the AUR. Use your preferred
AUR helper:

EXAMPLE

```sh
# with yay
yay -S qt5-webkit
```

**TIP:** To check whether `qt5-webkit` is already installed, run
`yay -Q qt5-webkit`.

### 2. Grab this project

EXAMPLE

```sh
# Clone the project folder to a scratch location, then copy it to the
# OS "tmp" folder:
cp -r ./globalprotect-ez /tmp/

# Go to the project folder:
cd /tmp/globalprotect-ez
```

### 3. Build and install the package

```sh
# Build the .pkg.tar.zst (fetches the upstream archive with -s)
makepkg -s

# Install
sudo pacman -U globalprotect-6.3.3.1-619-x86_64.pkg.tar.zst
```

Or in a single step:

```sh
makepkg -si
```

If `makepkg -s` fails to download the upstream archive automatically, see
[Download failure](#download-failure).

**NOTE:** `makepkg` fetches `PanGPLinux-6.3.3-c22.tgz` from the URL
specified in the `PKGBUILD` and verifies the SHA-256 checksum before building.

Then remove the scratch project folder:

```sh
rm -r /tmp/globalprotect-ez
```

### 4. Start the services

```sh
# System daemon (PanGPS) — handles the VPN connection itself
sudo systemctl enable --now gpd

# Verify it came up
sudo systemctl status gpd
```

### 5. Open the interface

Two components are started automatically on the next desktop login
(both via `/etc/xdg/autostart/*.desktop`):

- **PanGPA** — user agent that bridges CLI/UI and the `PanGPS` daemon.
  **Without it running, UI and CLI both fail with
  "Cannot connect to local gpd service"**.
- **PanGPUI** — the GUI.

**TIP:** To open without logging out (starts both in the current session)...

```sh
/opt/paloaltonetworks/globalprotect/PanGPA start &
sleep 5
globalprotect launch-ui
```

. If the `PanGPA start` command complains that the `gp_excluded_users.txt`
file is missing (`No such file or directory`), see
[Info: missing `gp_excluded_users.txt`](#info-missing-gp_excluded_userstxt).

---

## Useful commands

Beyond `connect`/`disconnect`/`show`, the official CLI exposes:

| Command | Purpose |
|---|---|
| `globalprotect help` | List all commands |
| `globalprotect` (no args) | Interactive prompt mode (`>>`); exit with `quit` |
| `globalprotect collect-log` | Generates a log bundle — **first step in troubleshooting** |
| `globalprotect set-log --level debug` | Increase verbosity |
| `globalprotect disable` | Turn off the VPN without uninstalling |
| `globalprotect rediscover-network` | Re-evaluate the network (useful after switching Wi-Fi, dock, etc.) |
| `globalprotect remove-user` | Clear saved credentials |
| `globalprotect resubmit-hip` | Resend the HIP report to the gateway |

Per-user configuration lives under `~/.globalprotect/`. To **reset the
app entirely** (wipes saved portals, credentials, preferences):

```sh
globalprotect disconnect
rm -rf ~/.globalprotect
```

---

## Optional configuration

### Corporate proxy

GlobalProtect only supports basic proxy configuration (no PAC and no
proxy authentication). For environments that require a proxy, edit
`/etc/environment`:

```sh
HTTP_PROXY="http://proxy.local:8080"
HTTPS_PROXY="https://proxy.local:8080"
NO_PROXY="localhost,127.0.0.1,*.internal.local"
```

`NO_PROXY` supports `*` wildcards from version 5.1.6 onward.

### Client certificate (mTLS)

If your portal requires certificate authentication, import the `.p12`
**before** connecting:

```sh
globalprotect import-certificate --location /path/to/client.p12
```

---

## Harmless log messages

### Info: missing `gp_excluded_users.txt`

On every PanGPA start you'll see a line like:

```
P...-T-... MM/DD/YYYY HH:MM:SS:mmm Info (  97): PanGPA: cannot open: /opt/paloaltonetworks/globalprotect//gp_excluded_users.txt, error: No such file or directory
```

It's tagged **Info**, not Error. `gp_excluded_users.txt` is an
optional, admin-controlled list of users to be *excluded* from having
the VPN enforced. When the file doesn't exist — the default for almost
every deployment — PanGPA logs this line and proceeds with "no
exclusions".

---

## Upgrading an existing install

Use this when a newer package has been built (either because upstream
released a new `PanGPLinux-*.tgz`, see *Packaging a new upstream
version* below, or because this project's `PKGBUILD` changed) and you
need to replace the installed version.

```sh
# 1. Stop the system daemon so files in use are released cleanly.
sudo systemctl stop gpd

# 2. Inside the project folder (with the new .pkg.tar.zst already built
#    via `makepkg -sf`), install the new package — pacman replaces the
#    old one in place:
sudo pacman -U globalprotect-<new-version>-x86_64.pkg.tar.zst

# 3. Bring the daemon back up.
sudo systemctl start gpd
```

The `.desktop` autostart files are refreshed by the install, but the
user-space processes (`PanGPA`, `PanGPUI`) currently in your session
are still running the old binaries. Either **log out and back in**, or
restart them in place:

```sh
pkill -x PanGPUI
pkill -x PanGPA
/opt/paloaltonetworks/globalprotect/PanGPA start &
sleep 5
globalprotect launch-ui
```

Per-user state in `~/.globalprotect/` (saved portals, credentials) is
preserved across upgrades.

### Reinstalling the same version

`pacman -U` refuses to install a package that already matches the
installed version. Either bump `pkgrel` in the `PKGBUILD` before
rebuilding, or force it:

```sh
sudo pacman -U --overwrite '*' globalprotect-6.3.3.1-619-x86_64.pkg.tar.zst
```

---

## Download failure

The `source=` URL in the `PKGBUILD` is a third-party mirror. If the
mirror is offline, moved, or blocked by a corporate firewall, `makepkg`
will abort with something like:

```
==> ERROR: Failure while downloading https://isssweb.umkc.edu/files/PanGPLinux-6.3.3-c22.tgz
```

**Workaround:** obtain the archive by any other means and drop it next
to the `PKGBUILD`. `makepkg` will detect the local file, skip the
download, and verify the checksum against what's declared in the
`PKGBUILD`.

Any of the following works:

1. Download directly with `curl` / `wget` (useful from a different
   network):

   ```sh
   curl -fLO https://isssweb.umkc.edu/files/PanGPLinux-6.3.3-c22.tgz
   ```

2. Mirror search — the archive is also mirrored by other Palo Alto
   partners. Search for the exact filename
   (`PanGPLinux-6.3.3-c22.tgz`).

3. Official download — requires a Palo Alto support account:
   <https://support.paloaltonetworks.com> → Updates > Software Updates
   → filter "GlobalProtect Agent for Linux".

Before rebuilding, verify the checksum matches:

```sh
sha256sum -c <<< 'e23ab15b813aaae577f19bfd61ef524b39d4bd5f55dd56c7db3076044978f04e  PanGPLinux-6.3.3-c22.tgz'
```

Then run `makepkg -s` again.

---

## Uninstall

```sh
sudo systemctl disable --now gpd
sudo pacman -R globalprotect
sudo systemctl restart systemd-resolved
```

Configuration under `~/.globalprotect/` is **not** removed by `pacman
-R`. Wipe it manually if you want a clean slate:

```sh
rm -rf ~/.globalprotect
```

---

## Upgrading to a newer upstream version

When Palo Alto releases a newer `PanGPLinux-*.tgz`, you can point this
project at it without waiting for anyone else to update the package.

**1.** Get the new archive. Drop it into the project folder (so you can
inspect it) — later you'll also need its URL, or the archive will have
to stay in place as a local file (see
[Download failure](#download-failure)).

```sh
cd /path/to/globalprotect-ez
curl -fLO <url-of-new-PanGPLinux-X.Y.Z-cNN.tgz>
```

**2.** Find the internal version of the UI RPM it ships. The two
numbers tell you what to put in `pkgver` and `pkgrel`:

```sh
tar -tzf PanGPLinux-*.tgz | grep -E 'UI_rpm.*\.rpm$' | grep -v _arm

# Sample output:
#   ./GlobalProtect_UI_rpm-6.3.3.1-619.rpm
#                            └─┬──┘ └┬┘
#                           pkgver  pkgrel
```

**3.** Compute the new checksum:

```sh
sha256sum PanGPLinux-*.tgz
```

**4.** Edit `PKGBUILD` and update these four lines:

```diff
-pkgver=6.3.3.1
-pkgrel=619
+pkgver=<new-pkgver>
+pkgrel=<new-pkgrel>

-_upstream_tgz="PanGPLinux-6.3.3-c22.tgz"
-_upstream_url="https://isssweb.umkc.edu/files/${_upstream_tgz}"
+_upstream_tgz="PanGPLinux-<X.Y.Z-cNN>.tgz"
+_upstream_url="<URL-hosting-the-new-tgz>"

-sha256sums=('e23ab15b813aaae577f19bfd61ef524b39d4bd5f55dd56c7db3076044978f04e')
+sha256sums=('<new-sha256>')
```

**5.** Regenerate `.SRCINFO`:

```sh
makepkg --printsrcinfo > .SRCINFO
```

**6.** Build and install as usual:

```sh
makepkg -s
sudo pacman -U globalprotect-<new-pkgver>-<new-pkgrel>-x86_64.pkg.tar.zst
```

> **Caveat — the `Ubuntu` binary patch.** The `prepare()` step replaces
> the 6-byte string `Ubuntu` inside the `PanGPS` binary with `Arch L`
> to bypass its distro check. Every release we've seen still ships that
> string, but if a future release drops or renames it, the daemon will
> refuse to start and the `prepare()` block will need to be revisited.
> Quick check on the new archive:
>
> ```sh
> bsdtar -xOf PanGPLinux-*.tgz ./GlobalProtect_UI_rpm-*.rpm \
>   | bsdtar -xO opt/paloaltonetworks/globalprotect/PanGPS \
>   | grep -c -a Ubuntu
> # Expected: 3 (or at least >0)
> ```

## About

globalprotect-ez 🄯 BSD-3-Clause  
Eduardo Lúcio Amorim Costa  
Brazil-ES  
https://www.linkedin.com/in/eduardo-software-livre/

![Brazil](./images/brazil.png)
