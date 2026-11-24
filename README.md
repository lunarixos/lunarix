# Lunarix

Lunarix is a minimal x86-64 Linux system with a BusyBox userspace. The live image
boots to a shell. The installed system boots from ext4 to a login on tty1, with
ttyS0 available for serial debugging.

## Build the live ISO

On an x86-64 glibc Linux host, install the Linux and BusyBox build tools
(GCC, binutils, make, Git, curl, flex, bison, bc, and libelf and OpenSSL development
headers), kmod, rsync, cpio, gzip, util-linux with sfdisk, e2fsprogs, xorriso,
squashfs-tools, CMake, mtools, GRUB with x86-64 EFI modules, tzdata, kbd with
console keymaps, and `localedef` with glibc locale definitions and UTF-8 character
maps. Also install the WPA supplicant tools (`wpa_supplicant`, `wpa_cli`, and
`wpa_passphrase`), OpenSSL, trusted CA
certificates, and the Realtek RTL8852AE firmware. Install zstd or xz if your host
compresses firmware with it. Then run:

```sh
scripts/build-iso
```

The result is `build/lunarix.iso`. The build downloads pinned Linux and BusyBox
sources when their local checkouts are missing. The system installer needs pkgr 0.5.3 or newer. Set `PKGR_WORKTREE` to
the updated adjacent checkout until that revision is published and pinned. Set `PKGR_SOURCE_DIR` for another checkout,
`LUNARIX_FIRMWARE_DIR` for a firmware directory other than `/lib/firmware`,
or `LUNARIX_CA_BUNDLE` for a CA file other than
`/etc/ssl/certs/ca-certificates.crt`. To use GRUB modules installed
outside `/usr/lib/grub/x86_64-efi`, set `GRUB_EFI_DIR`.

To build the ISO with local pkgr changes, point `PKGR_WORKTREE` at that
checkout. The live image and installed system track the version reported
by the compiled binary. The build also uses pkgr's Fastfetch recipe. To reuse
a previously downloaded Fastfetch source tarball, point `PKGR_SOURCE_CACHE`
at its directory.

```sh
PKGR_WORKTREE="$PWD/../pkgr" ./scripts/build-iso
```

Set `PKGR_SEED_DIR` to include additional trusted package archives on the ISO,
such as Git. The live repo reads these archives from the boot media, and the
installer copies them to the installed repo. Install any of them by name
with `pkgr install`. Set `LUNARIX_ISO` to choose the output path. For the
optional developer image, put the `base-devel`, `git`, and `make` archives
in the seed directory.

```sh
LUNARIX_ISO="$PWD/build/lunarix-devel.iso" PKGR_WORKTREE="$PWD/../pkgr" \
  PKGR_SEED_DIR="$PWD/../pkgr/build/repo" ./scripts/build-iso
```

## Install

1. Boot the ISO in x86-64 UEFI mode with Secure Boot disabled. Use **Lunarix Live
   (serial)** for a serial-only VM.
2. Run `lunarix-install`. Use the numbered setup menu to select a whole disk
   of at least 512 MiB, set the login username, and review the settings.
   The installer shows the partitions that will be erased and requires an
   explicit confirmation. Then set the user and root passwords.
3. Power off, remove the live media, and boot the installed disk through UEFI.
   Log in on tty1 or the serial console.

The installed system records these choices in `/etc/vconsole.conf`,
`/etc/hostname`, `/etc/locale.conf`, and `/etc/localtime`. The installer
copies the selected data from the ISO, so setup works offline.

## Boot in QEMU

Install QEMU with `qemu-system-x86_64` and `qemu-img`, and the x86-64 OVMF firmware.
From the repository root, open the live ISO in a QEMU window with:

```sh
./scripts/boot-iso
```

Run `lunarix-install` inside the guest and select `/dev/vda`. When it finishes,
close the QEMU window, or press **Ctrl+A**, then **X** in serial mode. Boot the
installed disk with:

```sh
./scripts/boot-iso --installed
```

Both commands attach the same virtual disk under `build/qemu/`. Live boots use
temporary UEFI settings and give the ISO boot priority. Set `LUNARIX_VM_DIR` to
use a separate virtual disk, for example:

```sh
LUNARIX_VM_DIR="$PWD/build/qemu-live" ./scripts/boot-iso
```

To erase the saved QEMU installation, shut down the VM and run
`./scripts/reset-vm`. It lists the virtual disk and saved UEFI settings it
will remove and asks you to type `RESET`. The next live boot creates a blank
disk. The reset command uses the same `LUNARIX_VM_DIR` override.
For terminal-only use, append `--serial` and select **Lunarix Live (serial)** when
booting the ISO. Run `ip link` to find the VirtIO interface, then use
`lunarix-net ethernet eth0` as root, substituting the actual interface name.
DHCP runs on demand.

## Networking and packages

The live ISO and installed system include `lunarix-net`, `pkgr`, and a CA
bundle for HTTPS. Run connection commands as root. Use `lunarix-net status`
to list the interfaces and routes.

```sh
lunarix-net ethernet eth0
lunarix-net wifi list
lunarix-net wifi connect 'My Wi-Fi'
lunarix-net wifi disconnect
pkgr --version
pkgr list
```

Replace `eth0` with your Ethernet interface. For Wi-Fi, the first supported
adapter is the Realtek RTL8852AE. Use `rfkill list` to check its radio state
if the adapter appears but scanning fails. If several Wi-Fi adapters are
present, append the interface name. The command prompts for a WPA2 or WPA3
Personal passphrase, obtains an IPv4 DHCP lease, and keeps credentials in
memory for the session. It starts the connection only when requested.
The base includes `pkgr` as a tracked package and ships the Fastfetch archive
in `/usr/share/pkgr/repo`. You can install it offline from the live shell or
a fresh install.

```sh
pkgr search fastfetch
pkgr install fastfetch
```

Put additional trusted archives in that directory, or install one directly
with `pkgr install ./FILE.pkgr.tar.gz`. You can select a signed online repository through `PKGR_REPO` and `PKGR_REPO_KEY`, then run
`pkgr sync`. The ISO comes with the local repository selected. Starting with
pkgr 0.4.0, `pkgr upgrade --all` updates installed packages from the selected
repository, and `pkgr upgrade NAME` updates one package. Both refresh the
catalog automatically when using HTTPS. `pkgr upgrade pkgr` uses the official
GitHub Release and needs a published release archive.

With the development tools installed, a regular user can build from a Git
checkout that contains a pkgr recipe.

```sh
lunarix-devel pkgr build https://github.com/lunarixos/pkgr.git packages/fastfetch
```

Install the resulting archive with pkgr as root.

To compare idle memory across builds, look at `MemAvailable` in
`/proc/meminfo` after login on the same VM or physical machine. To check disk
usage after installation, run `df -h /`.

## Network access as a regular user

New installations include sudo. The installer adds the first user to
`netadmin`, which may run `lunarix-net` as root after authenticating with the
user's own password.

```sh
sudo /usr/sbin/lunarix-net ethernet eth0
sudo /usr/sbin/lunarix-net wifi connect 'My Wi-Fi'
```

Administrators can edit the rule with `visudo`. The group grants access
to the networking command. Root logins are available for system administration.
To use OpenDoas instead, install the optional `opendoas` package and run
`doas /usr/sbin/lunarix-net` with the same arguments.

## Development tools

The optional `base-devel` package includes GCC, G++, headers, binutils,
CMake, Ninja, and common build tools. pkgr installs Git and GNU Make
automatically as dependencies with pkgr 0.3.0 or newer. The development
ISO provides all three archives for offline installation. As root in the
live shell or a fresh development-image install, run

```sh
pkgr install base-devel
make --version
pkgr install fastfetch
```

Run builds as your regular user with `lunarix-devel` to put the toolchain
on your path. The included Fastfetch recipe needs internet access to fetch
its pinned sources, or a source cache. To build it yourself, run
`lunarix-devel pkgr build fastfetch` and install the resulting archive as
root. If Fastfetch is already installed, `pkgr upgrade` expects a newer
package version.

```sh
lunarix-devel make CMAKE_BUILD_TYPE=Release
```

For development builds in QEMU, use the expanded CPU model and allow more
guest memory. From the Lunarix source checkout on the host:

```sh
LUNARIX_VM_CPU=max LUNARIX_VM_MEMORY=2G ./scripts/boot-iso --installed
```

To update an existing QEMU install, set `LUNARIX_ISO=$PWD/build/lunarix-devel.iso`
when running the installed-disk command above. This attaches the image as a
read-only CD. In the guest, run as root

```sh
mount -t iso9660 -o ro /dev/sr0 /mnt
pkgr upgrade /mnt/packages/pkgr-update.pkgr.tar.gz
pkgr --repo /mnt/packages install base-devel
```

The upgrade command is only needed if the installed pkgr is older. If you
want the development archives available after removing the media, copy
the Git, Make, and base-devel archives into `/usr/share/pkgr/repo`.

See [Rolling releases](rolling-release.md) for the base package layout, in-place
migration, and the signed repository build.

The original `scripts/build` and `scripts/boot` commands are available for direct
QEMU kernel boots during development.
