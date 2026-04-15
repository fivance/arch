# 🐧 Arch Linux Installation Guide

A step-by-step guide for installing Arch Linux from scratch, covering both BIOS and UEFI systems.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Verify Boot Mode](#verify-boot-mode)
- [Network Setup](#network-setup)
- [Partitioning](#partitioning)
- [Format Partitions](#format-partitions)
- [Mount Filesystems](#mount-filesystems)
- [Mirrors & Base Install](#mirrors--base-install)
- [fstab](#fstab)
- [Chroot & System Configuration](#chroot--system-configuration)
- [Users](#users)
- [Network](#network)
- [Bootloader](#bootloader)
- [Unmount & Reboot](#unmount--reboot)
- [Post-Install Extras](#post-install-extras)

---

## Prerequisites

Burn an ISO image to a USB drive and boot from it to start the Arch install.

---

## Verify Boot Mode

```bash
cat /sys/firmware/efi/fw_platform_size
```

| Output | Meaning |
|--------|---------|
| `64` | UEFI mode, 64-bit (x64) |
| `32` | UEFI mode, 32-bit (IA32) — limited bootloader support |
| No such file or directory | BIOS (or CSM) mode |

---

## Network Setup

```bash
ip link
```

### Wi-Fi (using iwctl)

```bash
iwctl
[iwd]# device list
[iwd]# device <name> set-property Powered on
[iwd]# adapter <adapter> set-property Powered on
[iwd]# station <name> scan
[iwd]# station <name> get-networks
[iwd]# station <name> connect <SSID>
```

### Verify Connection

```bash
ping archlinux.org
```

### Sync Time

```bash
timedatectl
```

---

## Partitioning

```bash
lsblk                  # list block devices
gdisk /dev/<device>    # e.g. gdisk /dev/sda
```

Inside `gdisk`, press `n` to create a new partition. Use the table below as a reference:

| Partition | Size | Hex Code | Notes |
|-----------|------|----------|-------|
| Boot | BIOS 2MB/ UEFI~1-2GB | `ef02` (BIOS) / `EF00` (UEFI) | Required for booting |
| Swap | ~1.5× your RAM | `8200` | Linux swap |
| Root | 20–50GB+ | `8300` | Linux filesystem |
| Home | Remaining space | `8300` | Linux filesystem |

> **Tip:** Press `p` inside gdisk to print and verify your partition layout, then `w` to write changes.

---

## Format Partitions

```bash
# Root and Home partitions
mkfs.ext4 /dev/sda3    # root
mkfs.ext4 /dev/sda4    # home

# Swap partition
mkswap /dev/sda2
swapon /dev/sda2
```

> **UEFI only:** also format the EFI/boot partition:
> ```bash
> mkfs.fat -F 32 /dev/sda1
> ```

Verify your work:

```bash
fdisk -l
```

---

## Mount Filesystems

```bash
mount /dev/sda3 /mnt          # root partition
mkdir /mnt/home
mount /dev/sda4 /mnt/home     # home partition
```

> **UEFI only:** mount the EFI system partition:
> ```bash
> mount --mkdir /dev/sda1 /mnt/boot
> ```

---

## Mirrors & Base Install

```bash
# Set up mirrors (adjust --country as needed)
reflector --country HR --latest 5 --sort rate --save /etc/pacman.d/mirrorlist

# Install base system
pacstrap -K /mnt base linux linux-firmware base-devel vim networkmanager
```

---

## fstab

Generate the fstab file to auto-mount partitions on boot:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab    # verify entries
```

> Expect 3 entries on BIOS systems and 4 on UEFI systems.

---

## Chroot & System Configuration

```bash
arch-chroot /mnt
```

### Timezone

```bash
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
# Example: ln -sf /usr/share/zoneinfo/US/Eastern /etc/localtime
hwclock --systohc
```

### Locale

```bash
vim /etc/locale.gen      # uncomment your locale (e.g. en_US.UTF-8 UTF-8)
locale-gen
vim /etc/locale.conf     # add: LANG=en_US.UTF-8
cat /etc/locale.conf     # verify
```

### Hostname

```bash
echo "yourhostname" >> /etc/hostname
cat /etc/hostname         # verify
```

---

## Users

```bash
passwd                              # set root password
useradd -m -G wheel,users <user>    # create user and add to groups
passwd <user>                       # set user password
```

---

## Network

```bash
systemctl enable NetworkManager
```

---

## Bootloader

```bash
pacman -S grub man-db man-pages

# BIOS systems:
grub-install --target=i386-pc /dev/sda

# UEFI systems (also install efibootmgr first):
pacman -S efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# Generate GRUB config (both):
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## Unmount & Reboot

```bash
exit          # exit chroot
umount -R /mnt
reboot
```

---

## Post-Install Extras

### Sudo for Non-Root Users

```bash
pacman -S vi
visudo        # uncomment the %wheel ALL=(ALL) ALL line
```

### Package Management

```bash
sudo pacman -Syu             # sync and update all packages
sudo pacman -Rns <package>   # remove package and unneeded dependencies
sudo pacman -Ss <query>      # search for a package
```

### Install AUR Helper (yay)

```bash
sudo pacman -S git
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
rm -rf yay/
```

### Manage Services

```bash
sudo systemctl status <service>    # check service status
sudo systemctl start <service>     # start a service
sudo systemctl stop <service>      # stop a service
# Example: sudo systemctl status NetworkManager
```

---

> **Note:** Replace placeholder values like `<device>`, `<user>`, `<SSID>`, and `<Region>/<City>` with your actual system details.
