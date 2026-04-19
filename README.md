# sg2002 Debian Image Recipe

This repository contains only the rootfs overlay and the manual steps to build
a bootable Debian image for the Milk-V Duo 256M. It does not use helper
scripts.

The image layout is:

- `p1`: FAT32, mounted as `/boot`
- `p2`: btrfs, mounted as `/`

The overlay lives in `rootfs-overlay.distro-packager/` and is copied into the
target rootfs during bootstrap.

## Requirements

Run everything as `root` on an amd64 Debian/Ubuntu host with these tools
installed:

```bash
apt-get install -y \
  mmdebstrap qemu-user-static binfmt-support \
  parted dosfstools btrfs-progs rsync
```

Your host must also be able to execute riscv64 binaries through `binfmt_misc`,
because package maintainer scripts are run during `mmdebstrap`.

## Build Image

From this repository:

```bash
set -euo pipefail

RECIPE_DIR=$PWD
IMG=$PWD/sg2002-duo256m-debian.img
ROOT_MNT=$PWD/mnt-root

install -d "$ROOT_MNT"

# Make the vendored Sodo repo key visible to host-side apt during bootstrap.
install -Dm0644 \
  "$RECIPE_DIR/rootfs-overlay.distro-packager/usr/share/keyrings/sodo-repo.gpg" \
  /usr/share/keyrings/sodo-repo.gpg

truncate -s 1536M "$IMG"
parted -s "$IMG" mklabel msdos
parted -s "$IMG" mkpart primary fat32 4MiB 260MiB
parted -s "$IMG" set 1 boot on
parted -s "$IMG" mkpart primary btrfs 260MiB 100%

LOOP=$(losetup --find --show --partscan "$IMG")
mkfs.vfat -F 32 -n BOOT "${LOOP}p1"
mkfs.btrfs -L rootfs -f "${LOOP}p2"

mount "${LOOP}p2" "$ROOT_MNT"
install -d "$ROOT_MNT/boot"
mount "${LOOP}p1" "$ROOT_MNT/boot"

mmdebstrap \
  --mode=root \
  --skip=check/empty \
  --architectures=riscv64 \
  --variant=minbase \
  --include='systemd-sysv,network-manager,netbase,kmod,btrfs-progs,ca-certificates,libbpf1,firmware-realtek,linux-image-cv181x-sodoport,u-boot-sg2002-milkv-duo256m-distroboot' \
  --setup-hook="sync-in $RECIPE_DIR/rootfs-overlay.distro-packager /" \
  --customize-hook='install -d "$1/boot/overlays"' \
  trixie \
  "$ROOT_MNT" \
  'deb https://deb.debian.org/debian trixie main non-free-firmware' \
  'deb https://deb.debian.org/debian trixie-updates main non-free-firmware' \
  'deb https://security.debian.org/debian-security trixie-security main non-free-firmware'

# Set the root password inside the foreign-arch rootfs without PAM.
printf 'root:CHANGE-ME\n' | chpasswd -c YESCRYPT -P "$ROOT_MNT"

# Run the normal kernel postinst hook one more time after bootstrap so DTBs are
# definitely synced into the mounted FAT /boot and extlinux is regenerated from
# the final installed state.
KVER=$(basename "$ROOT_MNT"/boot/vmlinuz-*)
KVER=${KVER#vmlinuz-}
chroot "$ROOT_MNT" /etc/kernel/postinst.d/zz-u-boot-menu "$KVER"

sync

umount "$ROOT_MNT/boot"
umount "$ROOT_MNT"
losetup -d "$LOOP"
```

## Notes

- The FAT `/boot` partition is mounted on `$ROOT_MNT/boot` before bootstrap, so
  `mmdebstrap` must be called with `--skip=check/empty`.
- Keeping `/boot` mounted from the start lets package maintainer scripts see a
  separate boot filesystem. The kernel package therefore installs
  `/vmlinuz-*`, `/initrd.img-*`, and regenerates `/boot/extlinux/extlinux.conf`
  through the normal Debian `u-boot-menu` hooks.
- Rerun `/etc/kernel/postinst.d/zz-u-boot-menu <version>` once after
  `mmdebstrap` completes. In the tested build this final explicit pass was
  still useful to force DTB syncing into `/boot/dtbs/<version>/`, after which
  `fdtdir /dtbs/<version>/` was generated correctly.
- The overlay is still copied via `--setup-hook`, which runs early enough for
  `/etc/kernel/cmdline`, `fstab`, and the APT repo configuration to influence
  package installation.
- `u-boot-sg2002-milkv-duo256m-distroboot` installs `fip.bin` directly into
  `/boot/`.
- `linux-image-cv181x-sodoport` generates `/boot/initrd.img-*` and
  `/boot/vmlinuz-*` through the normal Debian kernel hooks, and those hooks
  also refresh `extlinux.conf`.
- Use `chpasswd -c YESCRYPT -P "$ROOT_MNT"` for the root password step. That
  avoids PAM and works reliably for a riscv64 rootfs prepared on an amd64 host.
- The overlay adds the Sodo APT repo, the repo signing key, `fstab`,
  initramfs configuration, initramfs module list, and the kernel command line.
- `libbpf1` is included so `systemd` can enable its optional cgroup-BPF
  helpers when the installed kernel exposes the required BPF/BTF support.
