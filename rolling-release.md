# Lunarix rolling releases

Existing installations should update the system by running `pkgr upgrade --all`
from the installed OS. The same package format, ownership records and repository
work for applications and the system base. Updates run on demand.

## Implementation plan

1. Extend pkgr to track system paths and symbolic links, recognize essential
   packages, resolve version requirements, and preserve modified config files.
   Stage and validate a complete batch before committing it. Recover a pending
   batch at boot.
2. Install the base from packages on new systems. Adopt files on existing
   installations against archives reconstructed from their original installer
   media. Keep machine settings such as accounts, hostname and mount tables in
   the installer and on the machine.
3. Package the kernel and its matching modules. Stage EFI updates separately
   from root filesystem transactions. Keep the previous bootable kernel and
   loader available while testing the new ones.
4. Build a signed repository snapshot. Test installation, migration and an
   in-guest upgrade with a reboot before publishing a candidate.
5. Publish the archives and index at the configured HTTPS endpoint. Embed the
   endpoint and the public signing key in new images, and use the same trust
   configuration when migrating existing installations.

## What pkgr manages

| Package | Contents |
| --- | --- |
| `pkgr` | The manager and bundled source recipes |
| `lunarix-busybox` | BusyBox and its command links |
| `lunarix-base` | Init configuration, default login profile and OS identity |
| `lunarix-net` | Network commands, DHCP integration, WPA and HTTPS utilities |
| `sudo` | On-demand privilege delegation for network commands. OpenDoas is available as an optional package. |
| `lunarix-ca-certificates` | CA certificates for HTTPS |
| `lunarix-firmware` | Shipped device firmware and regulation data |
| `lunarix-boot` | EFI loader and the boot deployment command |
| `linux-lunarix` | Kernel image and matching module files |

The installer creates the filesystem layout and machine-specific configuration.
Kernel packages stage their content on the root filesystem. The boot
deployment command applies the kernel and loader to the configured EFI
partition and can be retried with `pkgr upgrade --all`.

## Build and serve an update

Build with the pkgr worktree at version 0.5.3 or newer. Bump
`config/system-release` when publishing a new package set. Give a kernel whose
modules have changed a new release string in `config/kernel/localversion`.
Build artifacts for the repository are produced under `build/system-repo`.

~~~sh
PKGR_WORKTREE="$PWD/../pkgr" ./scripts/build-iso
./scripts/build-repository /path/to/new-snapshot /path/to/private-signing-key.pem
~~~

Serve the new snapshot over HTTPS after it passes its upgrade test. During
promotion, keep previously indexed archive filenames accessible at the same
repository URL so clients can finish downloads from an earlier index. Do not publish the signing key. Set
`LUNARIX_REPO_URL` and `LUNARIX_REPO_KEY` during ISO construction to embed the
repository URL and its **public** key in new installs. On an existing install
the corresponding files are `/etc/pkgr/repository` and `/etc/pkgr/repo.pem`.

A dedicated GitHub repository such as `lunarixos/pkgr-repos` can host the
snapshots as release assets. Upload the signed index and each package archive
to the release. The stable client URL ends in `/releases/latest/download`.
Publish a complete draft release together and retain previously indexed
versioned archive names during transitions.

## Migrate an existing installation

Build the new image with `LUNARIX_LEGACY_ISO` pointing to the original ISO
used to install that system. Attach the new image as media, mount it at
`/mnt`, and run the migration as root inside the installed Lunarix system.

~~~sh
sh /mnt/migration/migrate
~~~

Migration verifies old binaries before registering their ownership and
allows locally edited files declared as configuration. It discovers the
EFI partition on the installed disk and keeps the old kernel as a boot menu
entry. An archive built from different installer media may fail to match
the old binaries.

## Verification

- Fresh x86-64 UEFI installation booted with packaged base components and
  working Ethernet DHCP.
- Migration from the saved 0.4.0 installer baseline preserved an edited
  profile and booted the new kernel.
- Signed HTTPS updates upgraded networking individually, then upgraded pkgr,
  the base packages and the kernel in one batch. The guest rebooted into
  `7.3.5-lunarix2`, then booted `7.3.5-lunarix1` from the fallback entry.
- pkgr's smoke tests exercise file ownership, configuration preservation,
  version requirements and recovery of an interrupted batch.

The repository and reboots were tested in disposable QEMU guests. Public
repository publication requires a production HTTPS endpoint and signing key.
