## Installation medium

Boot from Arch Linux archiso x86_64 UEFI CD

### Set the keyboard layout

```sh
# Temporarily set the keymap
$ loadkeys colemak
```

### UEFI mode

```sh
# Verify that the current mode is UEFI
$ efivar -l
$ ls /sys/firmware/efi/efivars
```

### Select the mirrors

```sh
# Select a regional mirror for best performance
$ nano /etc/pacman.d/mirrorlist
> http://cosmos.cites.illinois.edu/pub/archlinux/
```

### Internet connection

```sh
# Verify that there is an internet connection
$ ping google.com
```

### System clock

```sh
# Ensure that the system clock is accurate
$ timedatectl set-ntp true
```

## Pre-Installation

### Partition the drive (GPT)

```sh
# Identify the drive (e.g. sda)
$ lsblk

# Open a GPT partitioning tool (gdisk or parted)
$ gdisk /dev/sda

# Type 'p' to see an initial summary

# Type 'n' to create the first (boot) partition
# Use defaults for partition number and first sector
# Type in '+512M' for the last sector
# Use ef00 hex code (EFI System)

# Type 'n' to create the last (root) partition
# Use defaults for all values

# Type 'p' to see the final summary
# Type 'w' to write it to disk
```

### Format the partitions

```sh
# Format the EFI System partition as FAT32
$ mkfs.fat -F32 /dev/sda1

# Format the root partition as Linux ext4
$ mkfs.ext4 /dev/sda2
```

### Mount the partitions

```sh
# Mount the root partition
$ mount /dev/sda2 /mnt

# Mount the boot partition
$ mkdir -p /mnt/boot
$ mount /dev/sda1 /mnt/boot
```

## Installation

### Install the base packages

```sh
# Install Arch Linux
$ pacstrap -i /mnt base base-devel
```

### Configure the system

```sh
# Generate an fstab file
$ genfstab -p /mnt >> /mnt/etc/fstab
```

### Change root

```sh
# chroot into the new system
$ arch-chroot /mnt /bin/bash
```

### Swap file

To create a swap file:

```sh
# Create a swap file that is at least 2/5 the size of your physical RAM
$ fallocate -l 4G /swapfile

# Set the correct permissions
$ chmod 600 /swapfile

# Format the swap file
$ mkswap /swapfile

# Activate the swap file
$ swapon /swapfile

# Finally, edit /etc/fstab to add an entry for the swap file
$ nano /etc/fstab
> /swapfile none swap defaults 0 0
```

To remove a swap file:

```sh
# Turn off the swap file
$ swapoff -a

# Delete the swap file
$ rm -f /swapfile

# Finally, remove the relevant entry from /etc/fstab
```

### Locale

```sh
# Edit /etc/locale.gen and uncomment the line[s] that starts with 'en_US'
# TODO:
$ nano /etc/locale.gen

# Generate the new locales
$ locale-gen

# Create /etc/locale.conf with the following contents
$ nano /etc/locale.conf
> LANG=en_US.UTF-8

# Change the keyboard layout
$ nano /etc/vconsole.conf
> KEYMAP=colemak

# Select a time zone (use tzselect to get time zone)
$ ln -s /usr/share/zoneinfo/America/Chicago /etc/localtime

# Adjust the time skew and set the time standard to UTC
$ hwclock --systohc --utc
```

### Configure the network

```sh
# Set the hostname
$ nano /etc/hostname
> myhostname

# Append the hostname to entries in /etc/hosts, for example:
# 127.0.0.1	localhost.localdomain	localhost	myhostname
# ::1	localhost.localdomain	localhost	myhostname
$ nano /etc/hosts

# Get the ethernet device name
$ ip link

# Enable the dhcpcd service
$ systemctl enable dhcpcd@devicename.service

# Install SSH
$ sudo aura -S openssh

# Enable the on-demand ssh service
$ sudo systemctl enable sshd.socket
```

### Install the video driver (nvidia cards)

```sh
# Install nouveau driver (use nvidia for potentially better performance)
$ pacman -S xf86-video-nouveau
```

### Install a UEFI bootloader (GRUB)

```sh
# Create an initial RAM disk
$ mkinitcpio -p linux

# Install grub and efibootmgr
$ pacman -S grub efibootmgr

# Install GRUB UEFI to /boot/EFI/grub
$ grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Arch-GRUB --recheck

# Use grub-mkconfig to generate grub.cfg
$ grub-mkconfig -o /boot/grub/grub.cfg
```

### Unmount the partitions and reboot

```sh
# Set the root password
$ passwd

# Exit from the chroot environment
$ exit

# Unmount partitions manually as a safety measure
$ umount -R /mnt

# Reboot the computer
$ reboot
```

## Post-Installation

Installation is now complete, boot from the disk and log in as root

### Set up a user

```sh
# Create a user account
# TODO: change shell here
$ useradd --home-dir /home/jordi --create-home jordi

# Give it a password
$ passwd jordi

# Install sudo
$ pacman -S sudo

# TODO: get rid of this and use wheel instead
# Give the account access to sudo
# Add 'jordi ALL=(ALL) ALL' to /etc/sudoers
$ nano /etc/sudoers
> ##
> ## User privilege specification
> ##
> root ALL=(ALL) ALL
> jordi ALL=(ALL) ALL

# Log out of the root account and log in to the user account
$ exit
```

### Install Pacaur

```sh
# Install git
$ sudo pacman -S git

# Clone aura-bin from the AUR
$ git clone https://aur.archlinux.org/aura-bin.git

# Make and install the package
$ cd aura-bin
$ makepkg -sri
$ cd ..
$ sudo rm -r aura-bin
```

### Configure audio

```sh
# Install alsa-utils
$ pacaur -S alsa-utils
```

### Install zsh and ZIM

```sh
# Install zsh
$ pacaur -S zsh zsh-completions

# Make zsh the default shell
$ chsh -s /usr/bin/zsh

# Install ZIM
$ pacaur -S zsh-zim-git
```

### Install X

```sh
# Install the X window system and mesa (use nvidia-libgl for better performance)
$ pacaur -S xorg-server xorg-server-utils xorg-xinit

# Copy the default xinitrc
$ cp /etc/X11/xinit/xinitrc ~/.xinitrc

# Install polkit
$ pacaur -S polkit

# Set the X keymap
$ localectl --no-convert set-x11-keymap us,us pc104 colemak, grp:shifts_toggle
```

### Install i3

```sh
# Install i3 window manager
$ pacaur -S i3

# Append 'exec i3' to the end of ~/.xinitrc
$ nano ~/.xinitrc
> # twm &
> # xclock -geometry 50x50-1+1 &
> # xterm -geometry 80x50+494+51 &
> # xterm -geometry 80x20+494-0 &
> # exec xterm -geometry 80x66+0+0 -name login
>
> exec i3

# Install dmenu
$ pacaur -S dmenu2 j4-dmenu-desktop

# Install a terminal emulator
$ pacaur -S rxvt-unicode
```
