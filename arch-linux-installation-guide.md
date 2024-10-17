# Arch Linux Installation Guide

```txt
    _             _       _     _
   / \   _ __ ___| |__   | |   (_)_ __  _   ___  __
  / _ \ | '__/ __| '_ \  | |   | | '_ \| | | \ \/ /
 / ___ \| | | (__| | | | | |___| | | | | |_| |>  <
/_/   \_\_|  \___|_| |_| |_____|_|_| |_|\__,_/_/\_\

 ___           _        _ _       _   _                ____       _     _
|_ _|_ __  ___| |_ __ _| | | __ _| |_(_) ___  _ __    / ___|_   _(_) __| | ___
 | || '_ \/ __| __/ _` | | |/ _` | __| |/ _ \| '_ \  | |  _| | | | |/ _` |/ _ \
 | || | | \__ \ || (_| | | | (_| | |_| | (_) | | | | | |_| | |_| | | (_| |  __/
|___|_| |_|___/\__\__,_|_|_|\__,_|\__|_|\___/|_| |_|  \____|\__,_|_|\__,_|\___|

```

## 1. General Info

- ETA: < 1h for initial manual installation
- Guides:
  - [https://wiki.archlinux.org/title/Installation_guide](https://wiki.archlinux.org/title/Installation_guide)
  - [https://www.youtube.com/watch?v=FxeriGuJKTM](https://www.youtube.com/watch?v=FxeriGuJKTM)
  - [https://www.youtube.com/watch?v=YC7NMbl4goo](https://www.youtube.com/watch?v=YC7NMbl4goo)

## 2. Prepare installation media and reboot into the installer

### 2.1 Download and check ISO

- [download Arch ISO from a http mirror](https://archlinux.org/download/)
- [download signature from arch download page itself, not mirror](https://archlinux.org/iso/2024.06.01/archlinux-2024.06.01-x86_64.iso.sig)
- check using `gpg --keyserver-options auto-key-retrieve --verify archlinux-2024.06.01-x86_64.iso.sig` (this may show a warning "WARNING: The key's User ID is not certified with a trusted signature!" but that is OK as long as the signature itself is valid)

### 2.2 Insert and prepare the flash drive

- insert flash drive
- find out flash drive id `ls -l /dev/disk/by-id/usb-*` => eg. `../../sdb`
- use `lsblk` to check, and unmount any mounted partitions (eg. `umount /dev/sdb1`)
- switch to root for the next steps `sudo su`
- if you have a previous bootable ISO on the flash drive, to make the flash drive re-usable use `wipefs --all /dev/disk/by-id/usb-My_flash_drive` to remove the ISO 9660 file system signature
- repartition it using `fdisk /dev/sdX`
  - `o` to create a DOS partition table
  - `n` to create a new partition (`p` - primary, `1` - partition number, leave defaults for `First sector` and `Last sector` - takes all available space)
  - `w` to write chances and exit
- reformat it using `mkfs.fat -F 32 -n 'Arch' /dev/sdX1` (max 11 chars label)

### 2.3 Make an Arch ISO bootable flash drive

- copy the image to flash drive and WAIT a few minutes for it to completely finish: `cat path/to/archlinux-version-x86_64.iso > /dev/disk/by-id/usb-My_flash_drive` (use the actual device file eg. `usb-*-0\:0` not `usb-*-part1` etc.)
- run `sync` to ensures buffers are fully written to the device before you remove it
- NOTE: A lot of resources recommend using `dd` for the command above but that's mostly for historical reasons
- NOTE: After completing the Arch installation, if you want to make the flash drive re-usable follow the steps from 2.1, but you might want to keep it for administration purposed

### 2.4 Disable Secure Boot and enter the installation

- reboot, enter BIOS/UEFI and disable Secure Boot so the USB flash drive can boot (some Linux distribution installers work with Secure Boot, Arch does not)
- reboot, enter BIOS/UEFI and boot from flash drive to enter the Arch installation

- IMPORTANT: additional configuration is needed post-install to re-enable Secure Boot, otherwise GRUB won't boot the Arch installation if you just re-enable it from BIOS/UEFI. You can use the system fine without Secure Boot until you make the necessary configuration.

## 3. After booting inside the installer

### 3.1 Connect to WiFi

- use `ip link` to make sure you network interface is enabled and get its name (eg. `wlan0` or `wlp3s0`)
- use `iwctl` and `station <device-name> get-networks` to list available networks (`exit` to exit)
- use `iwctl --passphrase 'xxx' station <device-name> connect <network-name>` to connect
- use `ip addr show` and `ping archlinux.org` to verify the connection

- NOTE: `iwctl` (from the `idw` package) is used during instal, on an typical fully installed system prefer using `nmcli` (from the `networkmanager` package) together with `nm-applet` (from the `network-manager-applet`) and the `nm-connection-editor` package

### 3.2 Update system clock

- done automatically when connecting to internet, use `timedatectl` to ensure it's in sync

### 3.3 Setup SSH for a more convenient installation from a remote computer using copy-paste commands (optional)

- use `passwd` to add a root password on the target computer (for the live instal system, not the final system)
- ensure `sshd` is running using `systemctl status sshd`
- connect from the remote computer using `ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@archiso.local` (if both computers are NOT on the same network use the IP address of the target)

- NOTE: you can use `tmux` on the target computer (already installed) to create a session, so that if you get disconnected, you can reattach to the session after you reconnect from the remote computer

## 4. Disk partitioning

### 4.1 Securely wipe disk

- use `lsblk` to get the disk identifier (eg. `nvme0n1`)
- securely wipe the disk using `dd if=/dev/urandom of=/dev/nvme0n1 bs=4096 status=progress` (takes about 20min for a 500G nvme drive)

### 4.2 Create partitions

- use `fdisk /dev/nvme0n1` to start creating partitions:
  - use `p` to print current partitions information
  - use `g` to create an empty partition table
  - NOTE: press `Y` to confirm removing previous partitions signatures if asked bellow
    - `n`, `1`, `+4G`
    - `n`, `2`, accept defaults
  - use `t`, `L` to list and set the type for the partitions:
    - `t`, `1`, `L`, `1` or `uefi` (alias) - EFI System
    - `t`, `2`, `L`, `44` or `lvm` (alias) - Linux LVM
  - use `p` to review current partitions information
  - use `w` to write to disk

### 4.3 Format partitions

- NOTE: the /boot partition will be left unencrypted, as well as well as not setting up Secure Boot bellow (both valuable, but add additional overhead to the initial setup and can be done later)

- use `mkfs.fat -F32 /dev/nvme0n1p1` to format the EFI partition (will be mounted as /boot)
- use `cryptsetup luksFormat /dev/nvme0n1p2` to encrypt the second partition, type `YES` and enter a new password
- use `cryptsetup open /dev/nvme0n1p2 cryptlvm` to open the encrypted partition and map it to a name (cryptlvm = mapper name, can be anything)
- use `modprobe dm_mod` to make sure the device-mapper kernel module is loaded
- use `pvcreate /dev/mapper/cryptlvm` to create a physical volume and `pvscan` to inspect the result
- use `vgcreate volgroup0 /dev/mapper/cryptlvm` to create a volume group
- use `lvcreate -n lv_swap -L 96G volgroup0` to create a logical swap partition (this much swap allows suspend to disk)
- use `lvcreate -n lv_root -L 32G volgroup0` to create a logical root partition
- use `lvcreate -n lv_home -l +100%FREE volgroup0` to create a logical home partition
- use `lvreduce -L -256M volgroup0/lv_home` to allow some space after the last partition needed by `e2scrub`
- use `vgdisplay` and `lvdisplay` to inspect the result
- use `vgscan` and `vgchange -ay` to activate all volume groups
- format the logical volumes:
  - `mkswap /dev/volgroup0/lv_swap`
  - `mkfs.ext4 /dev/volgroup0/lv_root`
  - `mkfs.ext4 /dev/volgroup0/lv_home`

### 4.4 Mount partitions

- to mount swap use `swapon /dev/volgroup0/lv_swap` and `swapon -a` to mark it as available
- to mount root use `mount /dev/volgroup0/lv_root /mnt`
- to mount home use `mount --mkdir /dev/volgroup0/lv_home /mnt/home`
- to mount EFI as boot use `mount --mkdir /dev/nvme0n1p1 /mnt/boot`

## 5. Install Arch

### 5.1 Prepare the chroot environment and switch to it

- use `pacstrap -K /mnt base linux linux-firmware` to install the base package, the Linux kernel and common firmware
- generate fstab using `genfstab -U /mnt >> /mnt/etc/fstab` and `cat /mnt/etc/fstab` to verify it is correct
- chroot into the system using `arch-chroot /mnt`

### 5.2 Install additional kernels and minimal set of packages

- use `pacman -S linux-lts intel-ucode dosfstools e2fsprogs lvm2 cryptsetup grub efibootmgr os-prober networkmanager base-devel git unzip wget neovim pacman-contrib man-db man-pages`

- NOTE: full list of packages present in the [installer](https://geo.mirror.pkgbuild.com/iso/latest/arch/pkglist.x86_64.txt)

### 5.3 Configure system time

- set timezone using `ln -sf /usr/share/zoneinfo/Europe/Bucharest /etc/localtime` and set the hardware clock to UTC using `hwclock --systohc`
- setup NTP:
  - use `nvim /etc/systemd/timesyncd.conf` to set `NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org` and `FallbackNTP=0.pool.ntp.org 1.pool.ntp.org 0.fr.pool.ntp.org`
  - enable at boot time using `systemctl enable systemd-timesyncd.service`

### 5.4 Configure Localization

- use `nvim /etc/locale.gen` to uncomment `en_US.UTF-8 UTF-8` and run `locale-gen`
- use `nvim /etc/locale.conf` to create the locale file and add `LANG=en_US.UTF-8`

### 5.5 Configure Network

- use `nvim /etc/hostname` to add a hostname (eg. `arch-thinkpad-t480`)
- enable NetworkManager at boot time `systemctl enable NetworkManager`

### 5.6 Configure Users

- set root password using `passwd`
- add an additional user with root privileges:
  - `useradd -m -G wheel <username>`, `passwd <username>`
  - use `EDITOR=nvim && visudo` to safely open the `/etc/sudoers` file and uncomment `%wheel ALL=(ALL:ALL) ALL`, respectively add `Defaults passwd_timeout=0`, `Defaults timestamp_timeout=60` and `Defaults insults` to the end of the file

### 5.7 Regenerate initramfs

- use `nvim /etc/mkinitcpio.conf` to edit the mkinitcpio file:
  - on the `HOOKS` line add `keyboard` (before `block`), respectively `encrypt` and `lvm2` (between `block` and `filesystems`)
    - eg. `HOOKS=(base *udev* autodetect microcode modconf kms *keyboard* keymap consolefont block *encrypt* *lvm2* filesystems fsck)`
  - on the `MODULES` line add `atkbd` (common built-in laptop keyboard driver loaded early during system boot needed for entering the decryption password)
    - eg. `MODULES=(... atkbd)`
- recreate the initial initramfs image using `mkinitcpio -P` (look for `[encrypt]` and `[lvm2]` hooks in the output)

- NOTE: the default is a busybox-based initramfs (`udev`), not a systemd-based initramfs (`systemd`)

### 5.8 Configure Bootloader (GRUB)

- use `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB` to install the bootloader
- use `blkid /dev/nvme0n1p2` to get the UUID for setting `cryptdevice` bellow (eg. `cryptdevice=UUID=a6be283f-a793-4631-8f48-909f01b6c6c9:cryptlvm`):

```sh
blkid /dev/nvme0n1p2
/dev/nvme0n1p2: UUID="a6be283f-a793-4631-8f48-909f01b6c6c9" TYPE="crypto_LUKS" PARTUUID="768fe268-6665-42c5-a026-62669a7c01b4"
```

- use `nvim /etc/default/grub` to modify the bootloader options
  - edit the line `GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 cryptdevice=UUID=<uuid-from-above>:cryptlvm root=/dev/volgroup0/lv_root quiet"`
  - set `GRUB_DISABLE_SUBMENU=y`, `GRUB_DEFAULT=saved` and `GRUB_SAVEDEFAULT=true` if you have multiple kernels installed (eg. `linux` and `linux-lts`)
- regenerate the grub config using `grub-mkconfig -o /boot/grub/grub.cfg`

### 5.9 Exit chroot and reboot the system

- type `exit` to exit the chroot environment
- unmount all partitions using `umount -R /mnt`
- type `reboot`

## 6. Post-Install

- see Arch Linux Post Install document
- see [General Recommendations](https://wiki.archlinux.org/title/General_recommendations) and [List of Applications](https://wiki.archlinux.org/title/List_of_applications)

