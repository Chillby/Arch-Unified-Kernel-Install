# **_Arch Unified Kernel Install_**

Installing Arch Linux with a Unified Kernel and BTRFS Sub-volumes + Encryption

## Special Thanks

A special thanks to **_[Ataraxxia's Tutorial](https://github.com/Ataraxxia/secure-arch)_** in which I used as a base to construct this and adapt for the use with BTRFS with Subvolumes.

# **_Install_**

⚠️ _Be sure to backup any data on drives to prevent any data loss!!_ ⚠️

## Determining the right disk

```
$ lsblk
```

## Wiping Data

⚠️ **_This removes all data_** ⚠️ including any exisiting partition table

```bash
$ wipefs –a /dev/nvme0n1
```

### (Optional) Secure Erase of the Drive

Filling the disk up with garbage to make it harder to recover previous data (This will take a while)

```
$ dd if=/dev/urandom of=/dev/nvme0n1 bs=1M
```

## Partition Setup

| Size    | Type       | Mount Point |
| :------ | :--------- | :---------- |
| `650MB` | `EFI`      | /boot/efi   |
| `--GB`  | `cryptvol` | /           |

## Creating Partitions with Gdisk

```
$ gdisk /dev/nvme0n1
```

### Creating GPT Table

The letters are options within gdisk that allow you to run specific commands, follow each letter with the _ENTER key_ to confirm the action

**_O_** _(Make GPT Table)_ ➡️ **_Y_** _(Proceed)_

### Creating EFI Partition

➡️ **N** _(new partition)_

➡️ **_Enter_** _Default (Partition 1)_

➡️ **_Enter_** Default (First Sector)

➡️ **+650MB** _(Last Sector)_

➡️ **ef00** (EFI system partition)

### Creating the Main Partition

➡️ **N** _(new partition)_

➡️ **_Enter_** _Default (Partition 2)_

➡️ **_Enter_** _Default (First Sector)_

➡️ **_Enter_** _Default (Last Sector)_

➡️ **8309** (Linux LUKS partition)

### Write Changes to the Disk

➡️ **W** _(Write)_

➡️ **Y** _(Confirm)_

## Creating LUKS volume

```
$ cryptsetup luksFormat /dev/nvme0n1p2
```

➡️ Type "**YES**" in all caps

## Mounting CryptVolume with SSD Optimisations

```
$ cryptsetup open --perf-no_read_workqueue --perf-no_write_workqueue --persistent /dev/nvme0n1p2 cryptroot
```

## Format Partitions

```
$ mkfs.fat -F32 /dev/nvme0n1p1
```

```
$ mkfs.btrfs /dev/mapper/cryptroot
```

## Creating BTRFS Subvolumes

```
$ mount /dev/mapper/cryptroot /mnt
```

```
$ cd /mnt
```

```
$ btrfs subvolume create @
$ btrfs subvolume create @home
```

```
$ cd
```

```
$ umount /mnt -R
```

## Mounting Subvolumes

```
$ mount -o noatime,compress=zstd:1,subvol=@ /dev/mapper/cryptroot /mnt
```

```
$ mount --mkdir -o noatime,compress=zstd:1,subvol=@home /dev/mapper/cryptroot /mnt/home
```

## Mounting EFI Partition

```
$ mount --mkdir /dev/nvme0n1p1 /mnt/boot/efi
```

## Using Arch Scripts to create base system

💡 Change **intel-ucode** ➡️ **amd-ucode** if you are on an _AMD Processor_

_The Nano text editor is installed, change if desired_

```
$ pacstrap /mnt base base-devel linux linux-firmware sudo dracut sbsigntools binutils networkmanager btrfs-progs efibootmgr sbctl intel-ucode nano
```

## Setting the FSTAB

```
$ genfstab -U /mnt >> /mnt/etc/fstab
```

## Chrooting into the system

```
$ arch-chroot /mnt
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
$ chmod +x /usr/local/bin/dracut-*
```

```
$ mkdir /etc/pacman.d/hooks
```

### Install Hook

```
/etc/pacman.d/hooks/90-dracut-install.hook
```

```
[Trigger]
Type = Path
Operation = Install
Operation = Upgrade
Target = usr/lib/modules/*/pkgbase

[Action]
Description = Updating linux EFI image
When = PostTransaction
Exec = /usr/local/bin/dracut-install.sh
Depends = dracut
NeedsTargets
```

### Remove Hook

```
/etc/pacman.d/hooks/60-dracut-remove.hook
```

```
[Trigger]
Type = Path
Operation = Remove
Target = usr/lib/modules/*/pkgbase

[Action]
Description = Removing linux EFI image
When = PreTransaction
Exec = /usr/local/bin/dracut-remove.sh
NeedsTargets
```

### Send BLKID to Kernel Parameters

```
$ blkid -s UUID -o value /dev/nvme0n1p2 >> /etc/dracut.conf.d/cmdline.conf
```

```
/etc/dracut.conf.d/cmdline.conf
```

```
kernel_cmdline="rd.auto rd.luks.name=<THE UUID>=cryptroot root=/dev/mapper/cryptroot rootfstype=btrfs”
```

### Set Flags

```
/etc/dracut.conf.d/flags.conf
```

```
compress=”zstd”
hostonly=”no”
Add_drivers+=” i915 btrfs “
```

### Set System Clock

```
$ ln -sf /usr/share/zoneinfo/<Region>/<city> /etc/localtime
```

```
$ hwclock --systohc
```

### Set Locales

```
/etc/locale.gen
```

💡 _Uncomment the locales you want_

```
$ locale-gen
```

_Set your locale in this file similar to how I have in the below example for Australian English_

```
/etc/locale.conf
```

```
LANG=en_AU.UTF-8
```

### Set Hostname

_In here set a name for your computer_

```
/etc/hostname
```

```
Jason-PC
```

### Create your User

_Change <NAME> to your desired username_

```
$ useradd -m <NAME>
```

_Set a password for your user_

```
$ passwd <NAME>
```

### Set Root Password (Optional)

_Optionally set a password for ROOT however if you are going to use sudo I would recommend not setting a ROOT password_

```
$ passwd
```

### Sudo Configuration

_Replace <editor> with your editor you previously installed, in the case of using nano you would write:_

```
$ EDITOR=nano visudo
```

_Uncomment the line that contains_

```
%wheel ALL=(ALL) ALL
```

_Add your user to the wheel group replacing <NAME> with your user name_

```
$ usermod -aG wheel <NAME>
```

### Enable NetworkManager on Boot

```
$ systemctl enable NetworkManager
```

### Regenerate Initramfs

_We are going to regenerate our initramfs by using the hook we created earlier_

```
$ pacman -S linux
```

### Creating the Boot Entry

```
$ efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Arch Linux" --loader 'EFI\BOOT\bootx64.efi' --unicode
```

## SecureBoot Setup in new System, by [Ataraxxia](https://github.com/Ataraxxia/secure-arch/blob/main/00_basic_system_installation.md#secureboot)

### Booting into the BIOS

_Remove the arch installation usb from the system and reboot entering your BIOS (Typically by pressing DEL or F2)_

```
$ reboot
```

Now we are going to setup Secure Boot for your new Arch system. Firstly erase your existing keys for SecureBoot in your BIOS and ensure it is enabled.

Check if Secureboot is in a setup mode, this can be done with the following command.

```
$ sbctl status
```

You should have the proceeding output

```
Installed:      ✘ Sbctl is not installed
Setup Mode:     ✘ Enabled
Secure Boot:    ✘ Disabled
```

Create keys and sign binaries:

```
$ sbctl create-keys
```

```
$ sbctl sign -s /boot/efi/EFI/BOOT/bootx64.efi
```

Configure dracut to know where are signing keys:

```
/etc/dracut.conf.d/secureboot.conf
```

```
uefi_secureboot_cert="/usr/share/secureboot/keys/db/db.pem"
uefi_secureboot_key="/usr/share/secureboot/keys/db/db.key"
```

We also need to fix sbctl's pacman hook. Creating the following file will overshadow the real one:

```
/etc/pacman.d/hooks/sbctl.hook
```

```
[Trigger]
Type = Path
Operation = Install
Operation = Upgrade
Operation = Remove
Target = boot/*
Target = efi/*
Target = usr/lib/modules/*/vmlinuz
Target = usr/lib/initcpio/*
Target = usr/lib/**/efi/*.efi
[Action]
Description = Signing EFI binaries...
When = PostTransaction
Exec = /usr/bin/sbctl sign /boot/efi/EFI/BOOT/bootx64.efi
```

Enroll previously generated keys

```
$ bctl enroll-keys
```

Reboot the system. Enable only UEFI boot in BIOS and set BIOS password so the devil dogs won't turn off the setting. If everything went fine you should first of all, boot into your system, and then verify with sbctl or bootctl:

```
$ sbctl status
```

```
Installed:	✓ sbctl is installed
Owner GUID:	YOUR_GUID
Setup Mode:	✓ Disabled
Secure Boot:	✓ Enabled
```

🎉🎉🎉 **_Congratulations you now have a bootable Arch System with a Unified Kernel and BTRFS Sub-volumes + Encryption_** 🎉🎉🎉
