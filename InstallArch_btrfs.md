# 🐧 Arch Linux Installation Guide (btrfs)

A step-by-step guide for installing Arch Linux from scratch, covering both BIOS and UEFI systems.
Uses **btrfs** with subvolumes for snapshot support (Timeshift / snapper compatible).

---

## Table of Contents

* [Prerequisites](#prerequisites)
* [Verify Boot Mode](#verify-boot-mode)
* [Network Setup](#network-setup)
* [Partitioning](#partitioning)
* [Format Partitions](#format-partitions)
* [Create btrfs Subvolumes](#create-btrfs-subvolumes)
* [Mount Filesystems](#mount-filesystems)
* [Mirrors & Base Install](#mirrors--base-install)
* [fstab](#fstab)
* [Chroot & System Configuration](#chroot--system-configuration)
* [Users](#users)
* [Network & Services](#network--services)
* [Bootloader](#bootloader)
* [Unmount & Reboot](#unmount--reboot)
* [Post-Install Extras](#post-install-extras)

---

## Prerequisites

Burn an ISO image to a USB drive and boot from it to start the Arch install.

### Keyboard Layout

The default layout is US. To change it:

```
localectl list-keymaps        # list available keymaps
loadkeys <your-keymap>        # e.g. loadkeys hr (Croatian)
```

### Optional: SSH Into the Live ISO

If you prefer to install from another machine (e.g. to copy-paste commands):

```
passwd              # set a temporary root password for the ISO session
systemctl start sshd
ip addr show        # get the IP, then SSH in from another machine: ssh root@<ip>
```

---

## Verify Boot Mode

```
cat /sys/firmware/efi/fw_platform_size
```

| Output | Meaning |
| --- | --- |
| `64` | UEFI mode, 64-bit (x64) |
| `32` | UEFI mode, 32-bit (IA32) — limited bootloader support |
| No such file or directory | BIOS (or CSM) mode |

---

## Network Setup

```
ip link
```

### Wi-Fi (using iwctl)

```
iwctl
[iwd]# device list
[iwd]# device <n> set-property Powered on
[iwd]# adapter <adapter> set-property Powered on
[iwd]# station <n> scan
[iwd]# station <n> get-networks
[iwd]# station <n> connect <SSID>
```

### Verify Connection

```
ping archlinux.org
```

### Sync Time

```
timedatectl
```

---

## Partitioning

```
lsblk                  # list block devices
gdisk /dev/<device>    # e.g. gdisk /dev/sda
```

Inside `gdisk`, press `n` to create a new partition. Use the table below as a reference:

| Partition | Size | Hex Code | Notes |
| --- | --- | --- | --- |
| Boot | BIOS 2MB / UEFI ~1-2GB | `ef02` (BIOS) / `EF00` (UEFI) | Required for booting |
| Swap | ~1.5× your RAM | `8200` | Linux swap |
| Root | Remaining space | `8300` | Linux filesystem — **no separate home partition needed with btrfs subvolumes** |

> **Tip:** Press `p` inside gdisk to print and verify your partition layout, then `w` to write changes.
>
> With btrfs, root and home live on the same partition as separate subvolumes. No separate home partition is needed.

---

## Format Partitions

```
# Root partition — formatted as btrfs
mkfs.btrfs /dev/sda3

# Swap partition
mkswap /dev/sda2
swapon /dev/sda2
```

> **UEFI only:** also format the EFI/boot partition:
>
> ```
> mkfs.fat -F 32 /dev/sda1
> ```

Verify your work:

```
fdisk -l
```

---

## Create btrfs Subvolumes

Mount the root partition temporarily to create subvolumes:

```
mount /dev/sda3 /mnt
```

Create the subvolumes:

```
btrfs subvolume create /mnt/@          # root
btrfs subvolume create /mnt/@home      # home
btrfs subvolume create /mnt/@log       # /var/log
btrfs subvolume create /mnt/@cache     # /var/cache
btrfs subvolume create /mnt/@snapshots # snapshots (used by Timeshift/snapper)
```

> **Why these subvolumes?**
> Separating `/var/log` and `/var/cache` means snapshots of `@` stay small and clean — logs and package caches are excluded. `@snapshots` gives Timeshift a dedicated location.

Unmount before remounting with subvolume options:

```
umount /mnt
```

---

## Mount Filesystems

Mount each subvolume with recommended btrfs options:

```
# Root subvolume
mount -o subvol=@,noatime,ssd,compress=zstd,space_cache=v2,discard=async /dev/sda3 /mnt

# Home subvolume
mkdir -p /mnt/home
mount -o subvol=@home,noatime,ssd,compress=zstd,space_cache=v2,discard=async /dev/sda3 /mnt/home

# /var/log
mkdir -p /mnt/var/log
mount -o subvol=@log,noatime,ssd,compress=zstd,space_cache=v2,discard=async /dev/sda3 /mnt/var/log

# /var/cache
mkdir -p /mnt/var/cache
mount -o subvol=@cache,noatime,ssd,compress=zstd,space_cache=v2,discard=async /dev/sda3 /mnt/var/cache

# Snapshots
mkdir -p /mnt/.snapshots
mount -o subvol=@snapshots,noatime,ssd,compress=zstd,space_cache=v2,discard=async /dev/sda3 /mnt/.snapshots
```

> **Mount options explained:**
> - `noatime` — don't update access times on reads, reduces write overhead
> - `ssd` — enables SSD-specific optimisations in btrfs
> - `compress=zstd` — transparent compression, saves significant disk space
> - `space_cache=v2` — improved free space cache for better performance
> - `discard=async` — async TRIM, keeps SSD performance healthy without blocking I/O

> **UEFI only:** mount the EFI system partition:
>
> ```
> mount --mkdir /dev/sda1 /mnt/boot
> ```

---

## Mirrors & Base Install

```
# Set up mirrors (adjust --country as needed)
reflector --country HR --latest 5 --sort rate --save /etc/pacman.d/mirrorlist

# Install base system
pacstrap -K /mnt \
  base linux linux-firmware base-devel \
  vim networkmanager network-manager-applet \
  btrfs-progs grub-btrfs \
  openssh sudo git reflector \
  bluez bluez-utils \
  pipewire pipewire-pulse pipewire-jack alsa-utils sof-firmware \
  iptables-nft firewalld \
  acpid man-db man-pages
```

> **CPU microcode** — install the appropriate package for your CPU inside chroot (step below):
>
> ```
> pacman -S intel-ucode   # Intel CPUs
> pacman -S amd-ucode     # AMD CPUs
> ```
>
> Then regenerate GRUB config after installing: `grub-mkconfig -o /boot/grub/grub.cfg`

---

## fstab

Generate the fstab file to auto-mount partitions on boot:

```
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab    # verify entries
```

> You should see entries for `@`, `@home`, `@log`, `@cache`, and `@snapshots` — each with their subvolume mount options.
> BIOS systems expect 5 entries (+ swap); UEFI adds the boot partition for 6+.

---

## Chroot & System Configuration

```
arch-chroot /mnt
```

### Timezone

```
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
# Example: ln -sf /usr/share/zoneinfo/US/Eastern /etc/localtime
hwclock --systohc
```

### Locale

```
vim /etc/locale.gen      # uncomment your locale (e.g. en_US.UTF-8 UTF-8)
locale-gen
vim /etc/locale.conf     # add: LANG=en_US.UTF-8
cat /etc/locale.conf     # verify
```

### Console Keymap (persistent)

If you changed the keymap earlier in the live ISO, make it stick in the installed system:

```
echo "KEYMAP=<your-keymap>" >> /etc/vconsole.conf   # e.g. KEYMAP=hr
```

### Hostname

```
echo "yourhostname" >> /etc/hostname
cat /etc/hostname         # verify
```

---

## Users

```
passwd                              # set root password
useradd -m -G wheel,users <user>    # create user and add to groups
passwd <user>                       # set user password
```

---

## Network

Enable all essential services before rebooting:

```
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable sshd
systemctl enable firewalld
systemctl enable fstrim.timer      # periodic SSD TRIM (weekly)
systemctl enable reflector.timer   # auto-refresh mirrorlist
systemctl enable acpid             # ACPI event handling (power button, lid, etc.)
```

---

## Bootloader

```
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

```
exit          # exit chroot
umount -R /mnt
reboot
```

---

## Post-Install Extras

### Sudo for Non-Root Users

```
pacman -S vi
visudo        # uncomment the %wheel ALL=(ALL) ALL line
```

### Package Management

```
sudo pacman -Syu             # sync and update all packages
sudo pacman -Rns <package>   # remove package and unneeded dependencies
sudo pacman -Ss <query>      # search for a package
```

### Install AUR Helper (yay)

```
sudo pacman -S git
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
rm -rf yay/
```

### Manage Services

```
sudo systemctl status <service>    # check service status
sudo systemctl start <service>     # start a service
sudo systemctl stop <service>      # stop a service
# Example: sudo systemctl status NetworkManager
```

### Set Up Snapshots with Timeshift

Timeshift works great with the `@` / `@home` subvolume layout used in this guide. `timeshift-autosnap` automatically takes a snapshot before every pacman upgrade, and `grub-btrfs` adds snapshots to the GRUB boot menu so you can boot into one if something breaks.

```
yay -S timeshift timeshift-autosnap
```

Launch Timeshift, select **btrfs** as the snapshot type, and it will auto-detect your `@` subvolume. Schedule automatic snapshots (e.g. daily) for ongoing protection.

To make GRUB pick up snapshots automatically, patch the `grub-btrfsd` service to watch for new snapshots at runtime:

```
sudo systemctl edit --full grub-btrfsd
```

Replace the `ExecStart` line with:

```
ExecStart=/usr/bin/grub-btrfsd --syslog -t
```

Then rebuild the GRUB config:

```
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo systemctl enable --now grub-btrfsd
```

### zram (Compressed Swap in RAM)

A lightweight alternative to a swap partition. zram compresses data in RAM instead of writing to disk — it only activates when RAM is nearly full, and is much faster than disk swap.

```
sudo pacman -S zram-generator
```

Create `/etc/systemd/zram-generator.conf`:

```
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
```

Reboot or run `systemctl daemon-reload && systemctl start systemd-zram-setup@zram0` to activate.

> **Note:** If you added a swap partition during partitioning, you can keep both — zram will be preferred due to the higher `swap-priority`.

### Useful Packages

```
sudo pacman -S libreoffice-fresh calibre vlc localsend doublecmd-qt6 qbittorrent thunderbird spotify bitwarden thunar thunar-archive-plugin thunar-volman gvfs gvfs-mtp gvfs-smb tumbler fastfetch kitty neovim kdeconnect timeshift
yay -S sublime-text-4 obsidian brave-bin visual-studio-code-bin
```

---

> **Note:** Replace placeholder values like `<device>`, `<user>`, `<SSID>`, and `<Region>/<City>` with your actual system details.
