# Lunarix

Lunarix is a minimal x86-64 Linux system with a BusyBox userspace. The live image
boots to a shell. The installed system boots from ext4 to a login on tty1, with
ttyS0 available for serial debugging.

## Build the live ISO

On an x86-64 glibc Linux host, install the Linux and BusyBox build tools
(GCC, binutils, make, Git, curl, flex, bison, bc, and libelf and OpenSSL development
headers), kmod, rsync, cpio, gzip, util-linux with sfdisk, e2fsprogs, xorriso,
mtools, GRUB with x86-64 EFI modules, tzdata, kbd with console keymaps, and
`localedef` with glibc locale definitions and UTF-8 character maps. Then run:

```sh
scripts/build-iso
```

The result is `build/lunarix.iso`. The build downloads pinned Linux and BusyBox
sources when their local checkouts are missing. To use GRUB modules installed
outside `/usr/lib/grub/x86_64-efi`, set `GRUB_EFI_DIR`.

## Install

1. Boot the ISO in x86-64 UEFI mode with Secure Boot disabled. Use **Lunarix Live
   (serial)** for a serial-only VM.
2. Run `lunarix-install`. Select a whole disk of at least 512 MiB, then choose
   the console keyboard layout, hostname, locale, time zone, and login name.
   Enter accepts the defaults shown in brackets. Review the summary and
   confirm the disk to erase. Set passwords for the new account and root.
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

Both commands use the same virtual disk and UEFI settings under `build/qemu/`.
For terminal-only use, append `--serial` and select **Lunarix Live (serial)** when
booting the ISO. After login as root, run `ip link` to find the VirtIO interface
and `udhcpc -i eth0` (substituting the actual interface name) to bring it online.
DHCP runs on demand.

To compare idle memory across builds, look at `MemAvailable` in
`/proc/meminfo` after login on the same VM or physical machine. To check disk
usage after installation, run `df -h /`.

The original `scripts/build` and `scripts/boot` commands are available for direct
QEMU kernel boots during development.
