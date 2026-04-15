PREREQUISITES

Burn ISO image to na USB and start Arch install
# Verify the boot mode (BIOS/UEFI)
cat /sys/firmware/efi/fw_platform_size 
# If the command returns 64, the system is booted in UEFI mode and has a 64-bit x64 UEFI.
# If the command returns 32, the system is booted in UEFI mode and has a 32-bit IA32 UEFI. While this is supported, it will limit   the  boot loader choice to those that support mixed mode booting.
# If it returns No such file or directory, the system may be booted in BIOS (or CSM) mode.

# VERIFY NETWORK

ip link
# WIFI 
iwctl
[iwd]# device list
[iwd]# device name set-property Powered on
[iwd]# adapter adapter set-property Powered on
[iwd]# station name scan
[iwd]# station name get-networks
[iwd]# station name connect SSID
ping ping.archlinux.org

# Update system
timedatectl

# PARTITIONING

lsblk # list devices
gdisk /$devicename # ex. if your disk is named sda do gdisk /dev/sda

# Create New partiton (boot partition)
1)When in gdisk press n
2)First sector:default
3)Last sector:  #here you actually specify disk partition size
Hex code or GUID: ef02 (BIOS) or EF00 (UEFI)

# Create New partiton (swap partition)
# Generally SWAP partition is 1.5 times your RAM
1) gdisk -> press n for new partition again
2) First sector:default
3) Last sector: Size of your swap partition
4) Hex code or GUID: 8200 (8200 is code for Linux swap)

# Create New partition (root partition)
# Depends how much space you have + how many packages do you plan to install
1) gdisk -> press n for new partition again
2) First sector:default
3) Last sector: Size of your swap partition
4) Hex code or GUID: 8300 (8300 is code for Linux filesystem)


# Create New partition (home partition)
1) gdisk -> press n for new partition again
2) First sector:default
3) Last sector: default (because we are filing out all of the available space)
4) Hex code or GUID: 8300 (8300 is code for Linux filesystem)

# Check your partitions
 When in gdisk: press p (print your partitions)
 If you are satisfied -> press w (write)

# FORMAT PARTITIONS FOR THEIR FILESYSTEMS

mkfs.ext4 /dev/sda3 (sda3 is root partition)
mkfs.ext4 /dev/sda4 (sda4 is home partition)

mkswap /dev/sda2 (sda2 is swap partition)
swapon /dev/sda2

# If you are on BIOS you are done at this point, if you are UEFI do this command
mkfs.fat -F 32 /dev/sda1 (sda1 is boot partition)

# Check your work
fdisk -l

# MOUNTING THE FILESYSTEMS

mount /dev/sda3 /mnt (sda3 is our root partition)
mkdir /mnt/home
mount /dev/sda4 /mnt/home (sda4 is our home partition)

# For UEFI systems, mount the EFI system partition. For example:
mount --mkdir /dev/sda1 /mnt/boot


# MIRRORS

reflector --country HR --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
pacstrap -K /mnt base linux linux-firmware base-devel vim networkmanager

# FSTAB
# mount our mount points on boot

genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab ->>>>> check your mount points (At this point if BIOS you would have 3 entries and if UEFI 4 partitions)

# BOOT 
arch-chroot /mnt

# LOCALIZATION
ln -sf /usr/share/zoneinfo/yourzone -> for example for US: ln -sf /usr/share/zoneinfo/US/Eastern /etc/localtime (use TAB for completion to help yourself)

hwclock --systohc

vim /etc/locale.gen ->> uncomment your locale and save
locale-gen (to generate locales)
vim /etc/locale.conf -> add your LANG variable (ex. LANG=en_US.UTF8)
cat /etc/locale.conf -> for double checking

echo "$yourhostname" >> /etc/hostname
cat /etc/hostname -> for double checking

# USERS

passwd (and set your root password)
useradd -m -G wheel,users $user ($user = your preferred username)
passwd $user (set password)

# NETWORK

systemctl enable NetworkManager

# BOOTLOADER

pacman -S efibootmgr (for UEFI)
pacman -S grub man-db man-pages
grub-install --target=i386-pc /dev/sda (For BIOS systems)
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# UNMOUNT

umount -R /mnt 
reboot

# EXTRAS AFTER INSTALL AND REBOOT

# SUDO for non root users
pacman -S vi
visudo (change your users permissions for sudo access)

# Updates
sudo pacman -Syu (S = sync, y= refresh db, u= packages that need to be updated do update)
sudo pacman -Rns (Removes a package and removes unneeded dependencies)
sudo pacman -Ss (search for packages)

# Install AUR repository (yay command)
sudo pacman -S git
git clone https://aur.archlinux.org/yay.git
makepkg -si
rm-rf yay/

# Check services
sudo systemctl status/start/stop $servicename (ex. systemctl status NetworkManager)



