# sg2002 Debian Image Recipe

This repository contains only the rootfs overlay and the manual steps to build
a bootable Debian image for the Milk-V Duo 256M. It does not use helper
scripts.

The image layout is:

- `p1`: FAT32, `128 MiB`, mounted as `/boot`
- `p2`: btrfs, about `1.7 GiB`, mounted as `/`

The overlay lives in `rootfs-overlay.distro-packager/` and is copied into the
target rootfs during bootstrap.

## Requirements

Run everything as `root` on an amd64 Debian/Ubuntu host with these tools
installed:

```bash
apt-get install -y \
  mmdebstrap qemu-user-static binfmt-support wget \
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
BOOT_START_MIB=4
BOOT_SIZE_MIB=128
ROOT_SIZE_MIB=1741
BOOT_END_MIB=$((BOOT_START_MIB + BOOT_SIZE_MIB))
ROOT_END_MIB=$((BOOT_END_MIB + ROOT_SIZE_MIB))

install -d "$ROOT_MNT"

# Make the vendored Sodo repo key visible to host-side apt during bootstrap.
install -Dm0644 \
  "$RECIPE_DIR/rootfs-overlay.distro-packager/usr/share/keyrings/sodo-repo.gpg" \
  /usr/share/keyrings/sodo-repo.gpg

truncate -s "$((ROOT_END_MIB * 1024 * 1024))" "$IMG"
parted -s "$IMG" mklabel msdos
parted -s "$IMG" mkpart primary fat32 "${BOOT_START_MIB}MiB" "${BOOT_END_MIB}MiB"
parted -s "$IMG" set 1 boot on
parted -s "$IMG" mkpart primary btrfs "${BOOT_END_MIB}MiB" "${ROOT_END_MIB}MiB"

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
  --include='systemd-sysv,network-manager,netbase,kmod,btrfs-progs,ca-certificates,libbpf1,firmware-realtek,linux-image-cv181x-sodoport,linux-headers-cv181x-sodoport,u-boot-sg2002-milkv-duo256m-distroboot' \
  --setup-hook="sync-in $RECIPE_DIR/rootfs-overlay.distro-packager /" \
  --customize-hook='install -d "$1/boot/overlays"' \
  trixie \
  "$ROOT_MNT" \
  'deb https://deb.debian.org/debian trixie main non-free-firmware' \
  'deb https://deb.debian.org/debian trixie-updates main non-free-firmware' \
  'deb https://security.debian.org/debian-security trixie-security main non-free-firmware'

# Optional: add the external Radxa AIC8800 SDIO Wi-Fi DKMS package set used by
# boards such as the LicheeRV Nano. This is not needed for the Milk-V Duo 256M.
AIC_REL=5.0+git20260123.5f7be68d-4
AIC_URL_BASE="https://github.com/radxa-pkg/aic8800/releases/download/5.0%2Bgit20260123.5f7be68d-4"
AIC_DIR=$PWD/aic8800-debs
install -d "$AIC_DIR"
wget -O "$AIC_DIR/aic8800-firmware_${AIC_REL}_all.deb" \
  "$AIC_URL_BASE/aic8800-firmware_${AIC_REL}_all.deb"
wget -O "$AIC_DIR/aic8800-sdio-dkms_${AIC_REL}_all.deb" \
  "$AIC_URL_BASE/aic8800-sdio-dkms_${AIC_REL}_all.deb"
cp "$AIC_DIR"/*.deb "$ROOT_MNT/root/"
chroot "$ROOT_MNT" apt-get update
chroot "$ROOT_MNT" apt-get install -y \
  /root/aic8800-firmware_${AIC_REL}_all.deb \
  /root/aic8800-sdio-dkms_${AIC_REL}_all.deb
rm -f "$ROOT_MNT"/root/aic8800-*.deb

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
- The documented layout above creates a `128 MiB` FAT `/boot` partition and a
  `1741 MiB` btrfs root partition, for a total image size of `1873 MiB`
  starting from the `4 MiB` partition offset.
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
- `linux-headers-cv181x-sodoport` is included so out-of-tree DKMS modules can
  build against the packaged kernel flavour during image creation.
- The optional Radxa AIC8800 step installs `aic8800-firmware` and
  `aic8800-sdio-dkms` from the release assets at
  `5.0+git20260123.5f7be68d-4`. The DKMS package itself only depends on
  `dkms`, so the matching kernel headers still need to be present explicitly.
