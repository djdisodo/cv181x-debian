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
BOOT_SEED=$PWD/boot-seed

install -d "$ROOT_MNT" "$BOOT_SEED"

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

mmdebstrap \
  --mode=root \
  --architectures=riscv64 \
  --variant=minbase \
  --include='systemd-sysv,network-manager,netbase,kmod,btrfs-progs,ca-certificates,firmware-realtek,linux-image-cv181x-sodoport,u-boot-sg2002-milkv-duo256m-distroboot' \
  --setup-hook="sync-in $RECIPE_DIR/rootfs-overlay.distro-packager /" \
  --customize-hook='install -d "$1/boot/overlays"' \
  --customize-hook='chroot "$1" u-boot-update' \
  trixie \
  "$ROOT_MNT" \
  'deb https://deb.debian.org/debian trixie main non-free-firmware' \
  'deb https://deb.debian.org/debian trixie-updates main non-free-firmware' \
  'deb https://security.debian.org/debian-security trixie-security main non-free-firmware'

# Set the root password inside the foreign-arch rootfs without PAM.
printf 'root:CHANGE-ME\n' | chpasswd -c YESCRYPT -P "$ROOT_MNT"

# The kernel and U-Boot packages already populated $ROOT_MNT/boot while /boot
# was just a plain directory on the rootfs. Save those files, mount the real
# FAT boot partition on /boot, copy the payload across, then rerun the
# u-boot-menu hook so extlinux paths and fdtdir match a separate /boot.
rsync -aH "$ROOT_MNT/boot/" "$BOOT_SEED/"
mount "${LOOP}p1" "$ROOT_MNT/boot"
rsync -aH --delete "$BOOT_SEED/" "$ROOT_MNT/boot/"

KVER=$(basename "$BOOT_SEED"/vmlinuz-*)
KVER=${KVER#vmlinuz-}
chroot "$ROOT_MNT" /etc/kernel/postinst.d/zz-u-boot-menu "$KVER"

sync

umount "$ROOT_MNT/boot"
umount "$ROOT_MNT"
losetup -d "$LOOP"
```

## Notes

- Do not mount the FAT partition on `$ROOT_MNT/boot` before bootstrap, because
  `mmdebstrap` expects the target directory to be empty.
- After bootstrap, rerun `/etc/kernel/postinst.d/zz-u-boot-menu <version>` with
  the FAT partition mounted on `/boot`. That regenerates `extlinux.conf` with
  `linux /vmlinuz-*`, `initrd /initrd.img-*`, and `fdtdir /dtbs/<version>/`
  instead of rootfs-style `/boot/...` paths.
- `u-boot-sg2002-milkv-duo256m-distroboot` installs `fip.bin` directly into
  `/boot/`.
- `u-boot-update` generates `/boot/extlinux/extlinux.conf`.
- `linux-image-cv181x-sodoport` generates `/boot/initrd.img-*` and
  `/boot/vmlinuz-*` through the normal Debian kernel hooks.
- Use `chpasswd -c YESCRYPT -P "$ROOT_MNT"` for the root password step. That
  avoids PAM and works reliably for a riscv64 rootfs prepared on an amd64 host.
- The overlay adds the Sodo APT repo, the repo signing key, `fstab`,
  initramfs policy, and the kernel command line.
