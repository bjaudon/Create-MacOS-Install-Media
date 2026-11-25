# Create MacOS Bootable USB Media on Linux

This guide describes how to create bootable macOS or OS X USB media from Linux.
The process extracts Apple's installer files from `InstallMacOSX.dmg`, creates an
HFS+ volume on a USB drive, and copies the required recovery and installer files
to it.

For official download locations for older macOS releases, including Catalina,
Mojave, and High Sierra, see
[Mr. Macintosh: How to Download macOS Catalina, Mojave or High Sierra Full Installers](https://mrmacintosh.com/how-to-download-macos-catalina-mojave-or-high-sierra-full-installers/).
The `El Capitan` values shown below are examples; substitute the downloaded installer
filename and volume name for the macOS or OS X version you are creating.

> [!WARNING]
> The target disk will be completely erased. Back up any data on it first, identify
> it carefully with `lsblk`, and never set `TARGET_DISK` to a disk containing the
> host operating system or another important volume.

## Requirements

- A Linux system
- A USB flash drive or external disk with at least 8 GB of capacity
- Root/sudo access
- Internet access
- `curl`, `dmg2img`, `7z`, HFS+ utilities, `parted`, and `losetup`
- A Mac compatible with the resulting macOS or OS X installer media

The commands below use OS X El Capitan as a working example. The general extraction
and USB preparation process may differ for newer macOS installers.

## 1. Install dependencies

### Fedora

```bash
sudo dnf install -y dmg2img p7zip p7zip-plugins hfsplus-tools parted curl
```

### Debian/Ubuntu

```bash
sudo apt update
sudo apt install -y dmg2img p7zip-full hfsprogs parted curl
```

## 2. Prepare a working directory

All downloads and intermediate disk images are placed outside the USB target.

```bash
WORK_DIR=~/Downloads/macos-installer
mkdir -p "$WORK_DIR"
cd "$WORK_DIR"
```

## 3. Download the MacOS installer

Download the installer for the macOS or OS X version you need. Use the example URL below for El Capitan, or replace it with another official installer URL.

```bash
curl -fLO "http://updates-http.cdn-apple.com/2019/cert/061-41424-20191024-218af9ec-cf50-4516-9011-228c78eda3d2/InstallMacOSX.dmg"
```

Confirm the download exists and is nonempty:

```bash
ls -lh InstallMacOSX.dmg
```

## 4. Extract the installer package

Convert the outer DMG to a raw disk image:

```bash
dmg2img InstallMacOSX.dmg InstallMacOSX.img
```

Extract the raw disk image and the HFSX partition:

```bash
mkdir -p outer_dmg hfs_extracted
7z x InstallMacOSX.img -o"./outer_dmg"
7z x "./outer_dmg/1.disk image.hfsx" -o"./hfs_extracted"
```

If the extracted HFSX partition has a different filename, list it with
`find outer_dmg -type f` and adjust the path in the final command above.

Locate the installer package and extract it:

```bash
PKG_PATH=$(find ./hfs_extracted -name "InstallMacOSX.pkg" -print -quit)
test -n "$PKG_PATH"
mkdir -p pkg_extracted
7z x "$PKG_PATH" -o"./pkg_extracted"
```

Locate `InstallESD.dmg` and convert it:

```bash
ESD_PATH=$(find ./pkg_extracted -name "InstallESD.dmg" -print -quit)
test -n "$ESD_PATH"
dmg2img "$ESD_PATH" InstallESD.img
```

## 5. Mount the installer disk image

Create mount points and attach `InstallESD.img` to a loop device:

```bash
sudo mkdir -p /mnt/install_esd /mnt/base_src /mnt/mac_usb
LOOP_ESD=$(sudo losetup --partscan --show --find InstallESD.img)
lsblk "$LOOP_ESD"
```

The El Capitan ESD image normally exposes its HFS+ volume as partition 2.
Mount that partition:

```bash
sudo mount -t hfsplus -o ro "${LOOP_ESD}p2" /mnt/install_esd
ls /mnt/install_esd
```

If the loop device has no `p2` partition on some systems, use the partition
listing from `lsblk` and replace `p2` with the correct partition name.

## 6. Extract and mount the base system

Convert `BaseSystem.dmg` from the mounted ESD image:

```bash
dmg2img /mnt/install_esd/BaseSystem.dmg BaseSystem.img
```

Attach the resulting disk image to another loop device:

```bash
LOOP_BASE=$(sudo losetup --partscan --show --find BaseSystem.img)
lsblk "$LOOP_BASE"
```

The base system is normally its partition 1. Mount it read-only:

```bash
sudo mount -t hfsplus -o ro "${LOOP_BASE}p1" /mnt/base_src
ls /mnt/base_src
```

As with the ESD image, use the partition name reported by `lsblk` if it differs.

## 7. Identify and erase the USB disk

List available disks and partitions:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Set `TARGET_DISK` only to the whole USB disk, not to a partition. Common examples
are `/dev/sda`, `/dev/sdb`, or `/dev/nvme0n1`.

```bash
TARGET_DISK="/dev/sda"
```

If your disk name ends in a number, such as `/dev/nvme0n1`, partition 1 is named
`p1`; otherwise it is named `1`, such as `/dev/sda1`. The following command handles
both cases:

```bash
if [[ "$TARGET_DISK" =~ [0-9]$ ]]; then
  TARGET_PART="${TARGET_DISK}p1"
else
  TARGET_PART="${TARGET_DISK}1"
fi

test -b "$TARGET_DISK"
printf 'Target disk: %s\nTarget partition: %s\n' "$TARGET_DISK" "$TARGET_PART"
```

Double-check the disk's size and make sure it is the intended USB device.

Unmount any currently mounted partitions on the target, erase its filesystem
signatures, create a GPT partition table, and format the partition as HFS+:

```bash
sudo umount "${TARGET_DISK}"* 2>/dev/null || true
sudo wipefs -a "$TARGET_DISK"
sudo parted -s "$TARGET_DISK" mklabel gpt
sudo parted -s "$TARGET_DISK" mkpart primary hfs+ 1MiB 100%
sudo mkfs.hfsplus -v "Install OS X El Capitan" "$TARGET_PART"
```

`mkfs.hfsplus` may take some time for larger USB disks.

## 8. Copy the installer to the USB disk

Mount the new HFS+ volume:

```bash
sudo mount -t hfsplus -o force,rw "$TARGET_PART" /mnt/mac_usb
```

Copy the base system:

```bash
sudo cp -rp /mnt/base_src/. /mnt/mac_usb/
```

Replace the package directory in the base system with the complete El Capitan
installer packages:

```bash
sudo rm -rf "/mnt/mac_usb/System/Installation/Packages"
sudo cp -rp /mnt/install_esd/Packages "/mnt/mac_usb/System/Installation/"
```

Copy the base-system disk image and chunklist:

```bash
sudo cp -p \
  /mnt/install_esd/BaseSystem.dmg \
  /mnt/install_esd/BaseSystem.chunklist \
  /mnt/mac_usb/
```

Verify the copied files and available space:

```bash
ls -lh /mnt/mac_usb/BaseSystem.dmg /mnt/mac_usb/BaseSystem.chunklist
df -h /mnt/mac_usb
```

## 9. Flush and unmount

Flush writes before disconnecting the disk:

```bash
sync
sudo umount /mnt/mac_usb
sudo umount /mnt/base_src
sudo umount /mnt/install_esd
sudo losetup -d "$LOOP_BASE"
sudo losetup -d "$LOOP_ESD"
```

Remove the temporary mount points:

```bash
sudo rmdir /mnt/mac_usb /mnt/base_src /mnt/install_esd
```

## 10. Boot the installer from a Mac

1. Safely eject the USB disk from Linux.
2. Connect it to the target Mac.
3. Start or restart the Mac while holding the **Option** key.
4. Select **Install OS X El Capitan** from the startup disk menu.
5. Follow Apple's installer prompts.

Network recovery or firmware restrictions may prevent older media from booting on
some Macs. In that case, verify that the Mac supports El Capitan and has not been
configured to reject external startup volumes.

## Troubleshooting

### `mount: wrong fs type`

Install or verify the HFS+ tools, then confirm the partition with:

```bash
sudo fdisk -l "$LOOP_ESD"
sudo fdisk -l "$LOOP_BASE"
lsblk -f
```

### The loop device partition name is not `p2` or `p1`

Use the exact child partition name printed by `lsblk "$LOOP_ESD"` or
`lsblk "$LOOP_BASE"`:

```bash
sudo mount -t hfsplus -o ro /dev/loop0p2 /mnt/install_esd
```

Replace `/dev/loop0p2` with the value that applies to the current loop device.

### `TARGET_DISK` is wrong

Stop before step 8. Re-run `lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS` and set
`TARGET_DISK` to the USB disk. Reformatting a disk is destructive and cannot be
undone.

### Temporary mount cleanup fails

Check only the mount points created by this guide:

```bash
findmnt /mnt/mac_usb
findmnt /mnt/base_src
findmnt /mnt/install_esd
```

Unmount only active mount points created by this guide:

```bash
sudo umount /mnt/mac_usb 2>/dev/null || true
sudo umount /mnt/base_src 2>/dev/null || true
sudo umount /mnt/install_esd 2>/dev/null || true
```

Detach only the loop devices created earlier in this guide. If the variables are
still available in the current shell:

```bash
sudo losetup -d "$LOOP_BASE"
sudo losetup -d "$LOOP_ESD"
```

Do not detach arbitrary loop devices. If those variables are no longer available,
identify their backing files and detach the matching devices explicitly:

```bash
sudo losetup -j "$(pwd)/BaseSystem.img"
sudo losetup -j "$(pwd)/InstallESD.img"
```

Use the exact loop devices printed by those commands only if they were created
from the corresponding images:

```bash
sudo losetup -d /dev/loopN
sudo losetup -d /dev/loopM
```

Replace `/dev/loopN` and `/dev/loopM` with the exact devices shown by
`losetup -j`; do not detach unrelated loop devices.
