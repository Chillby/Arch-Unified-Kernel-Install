
# Arch Unified Kernel Install

Installing Arch Linux with Unified Kernel and BTRFS Sub-volumes + Encryption

⚠️ *Be sure to backup any data on drives to prevent any data loss!!* ⚠️ 

## Determining the right disk

```
lsblk
```
## Wiping Data

⚠️ ***This removes all data*** ⚠️ including any exisiting partition table

```bash
wipefs –a /dev/nvme0n1
```
## Partition Setup

| Size | Type     | Mount Point                |
| :-------- | :------- | :------------------------- |
| `650MB` | `EFI` | /boot/efi |
| `--GB` | `cryptvol` | /

## Creating Partitions with Gdisk

```
gdisk /dev/nvme0n1
```

### Creating GPT Table
The letters are options within gdisk that allow you to run specific commands, follow each letter with the *ENTER key* to confirm the action

***O*** *(Make GPT Table)* ➡️ ***Y*** *(Proceed)*

### Creating EFI Partition

➡️ **N** *(new partition)*

➡️ ***Enter*** *Default (Partition 1)*

➡️ ***Enter*** Default (First Sector)

➡️ **+650MB** *(Last Sector)*

➡️ **ef00** (EFI system partition)

### Creating the Main Partition

➡️ **N** *(new partition)*

➡️ ***Enter*** *Default (Partition 2)*

➡️ ***Enter*** *Default (First Sector)*

➡️ ***Enter*** *Default (Last Sector)*

➡️ **8309** (Linux LUKS partition)

### Write Changes to the Disk

➡️ **W** *(Write)*

➡️ **Y** *(Confirm)*

## Creating LUKS volume

```
cryptsetup luksFormat /dev/nvme0n1p2
```

➡️ Type "**YES**" in all caps

## Mounting CryptVolume with SSD Optimisations

```
cryptsetup open --perf-no_read_workqueue --perf-no_write_workqueue --persistent /dev/nvme0n1p2 cryptroot
```

## Format Partitions

```
mkfs.fat -F32 /dev/nvme0n1p1
```
```
mkfs.btrfs /dev/mapper/cryptroot
```

## Creating BTRFS Subvolumes
```
mount /dev/mapper/cryptroot /mnt
```
```
cd /mnt
```
```
btrfs subvolume create @
btrfs subvolume create @home
```
```
cd
```
```
umount /mnt -R
```

## Mounting Subvolumes

```
mount -o noatime,compress=zstd:1,subvol=@ /dev/mapper/cryptroot /mnt
```
```
mount --mkdir -o noatime,compress=zstd:1,subvol=@home /dev/mapper/cryptroot /mnt/home
```

## Mounting EFI Partition

```
mount --mkdir /dev/nvme0n1p1 /mnt/boot/efi
```

## Using Arch Scripts to create base system

💡 Change **intel-ucode** ➡️ **amd-ucode** if you are on an *AMD Processor*

*The Nano text editor is installed*
```
pacstrap /mnt base base-devel linux linux-firmware intel-ucode dracut sbsigntools binutils networkmanager btrfs-progs efibootmgr sbctl nano
```

## Setting the FSTAB

```
genfstab -U /mnt >> /mnt/etc/fstab
```

## Chrooting into the system

```
arch-chroot /mnt
```

### Auto Dracut Update on kernel upgrade

```
/usr/local/bin/dracut-install.sh
```
```shell
#!/usr/bin/env bash

mkdir -p /boot/efi/EFI/BOOT

while read -r line; do
 if [[ "$line" == 'usr/lib/modules/'+([^/])'/pkgbase' ]]; then
  kver="${line#'usr/lib/modules/'}"
  kver="${kver%'/pkgbase'}"

  dracut --force --uefi --kver "$kver" /boot/efi/EFI/BOOT/bootx64.efi
 fi
done
```

### Auto Dracut Remove on kernel upgrade
```
/usr/local/bin/dracut-remove.sh
```
```shell
#!/usr/bin/env bash
rm -f /boot/efi/EFI/BOOT/bootx64.efi
```
## Creating Pacman Hooks

```
chmod +x /usr/local/bin/dracut-*
```
```
mkdir /etc/pacman.d/hooks
```

### Install Hook
