# Arch Linux on UEFI Installation Guide
## Preparation 

### Prepare an installation medium
1. Download the latest release of  [Ventoy](https://github.com/ventoy/Ventoy/releases/)  (or any other bootable meida creation tool, we use ventoy as an example here) to create a bootable USB flash drive, with no less than 4G storage. Make sure there is no important data in it or you already have a backup.
2. Download installation image of [Arch Linux](https://archlinux.org/download/), the archive name should look like `archlinux-YYYY.MM.DD-x86_64.iso` 
3. Copy the installation image into partation named `Ventoy`

### BIOS configuration
1. Hold F2/ESC/Del etc.(depends on your device) during startup to access bios.
2. Make sure UEFI is Enabled.
3. Disable Secure Boot.
4. Disable Fast Startup Mode.
5. Adjust boot order. Check if there is your UEFI USB flash drive and move it upto the first place of the boot order.

## Establish environment of Installation
After completing the BIOS settings, restart the computer. You should now enter the Ventoy bootloader interface. Select `archlinux-YYYY.MM.DD-x86_64.iso` and **boot in normal mode** to enter the installation program. 

### Set the console keyboard layout and font
The default console keymap is US. Available layouts can be listed with:
```
# ls /usr/share/kbd/keymaps/**/*.map.gz
```
To set the keyboard layout, pass a corresponding file name to loadkeys, omitting path and file extension. For example, to set a Spanish keyboard layout, run:
```
# loadkeys es
```
Console fonts are located in /usr/share/kbd/consolefonts/ and can likewise be set with setfon. For example, to use one of the largest fonts suitable for HiDPI screens, run:
```
# setfont ter-132b
```
You can reset the font with:
```
# setfont
```

### Establish an Internet Connection
The most reliable way is to use a wired connection, as Arch is setup by default to connect to DHCP. In this way you can just test your connection by:
```
# ping archlinux.org
```
> you can use `Ctrl+C` to terminate a running program.

However, you can also use wireless networks by following these steps:
1. Unblock any possible hardware or software block
```
# rfkill unblock all
```
2. List network devices
```
# ip link show
```
3. Check the status of wireless network device (generally, it is named as `wlan0`, if not, please replace `wlan0` with the name of the network device shown in your device), it needs to be set as `UP`.
```
# ip link set wlan0 up
```
4. Use iwctl to connect to the wireless network.
```
# iwctl
```
5. Scan available networks, and print the list
```
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
```
6. Connect your device to the target network (replace `WiFi-SSID` with your target network), then enter passphrase.
```
[iwd]# station wlan0 connect Wifi-SSID
```
>> You can also use this combined command to connect to network.
>> __# iwctl --passphrase `passphrase` station device connect `Wifi-SSID`__
7. Use `quit` to terminate `iwc`, and verify your connection.
```
# ping archlinux.org
```

### Update the system clock
```
# timedatectl set-ntp true
```

### Partition the disks
1. List all storage devices and partitions.
```
# fdisk -l
```
2. Select target disk `/dev/nvme0n1` (subject to your device).
```
# fdisk /dev/nvme0n1
```

#### Create EFI system partition
1. Use `n` to create a new partition, leave first sector as default setting and `+512M` as second sector (recommended).
2. Use `t` to mark the new partition as `EFI System` (to list all known partition type with command `L`). Typically the partition type ID of EFI System would be `1`.
3. Use `w` to save and quit fdisk.

#### Format EFI system partition
```
# mkfs.fat -F32 /dev/nvme0n1p1
```

#### Create file system partition
1. Use `n` to create a new partition, let all settings as default (recommended).
2. Use `w` to save and quit fdisk.

#### Format file system partition
```
# mkfs.ext4 /dev/nvme0n1p2
```
> Replace `nvme0n1p2` with your file system partition.

### Mount partitions
> Replace __nvme0n1p2__ with your file system partition，__nvme0n1p1__ with your EFI system partition.
```
# mount /dev/nvme0n1p2 /mnt
```
```
# mkdir /mnt/boot
# mount /dev/nvme0n1p1 /mnt/boot 
```
Verify the mounted partitions
```
# mount
```

## Installation

### Select the mirrors
> Replace `Spain` with your location, this command would find the fastest mirror of the location. All of the shown `warning` are ignorable.
```
# reflector --country Spain --sort rate --latest 5 --save /etc/pacman.d/mirrorlist
```

### Enable ParallelDownloads
> Open the configuration file of `pacman`, and uncomment `ParallelDownloads = 5`.
```
# vim /etc/pacman.conf
```

### Install essential packages
> Use `enter` to skip and leave the options as default. (recommended)
```
# pacstrap /mnt base base-devel linux linux-firmware dhcpcd vim reflector
```

## Configure the system

### Generate Fstab (File System Table)
```
genfstab -L /mnt >> /mnt/etc/fstab
```
> Check the resulting `/mnt/etc/fstab` file, and edit it in case of errors.
```
# cat /mnt/etc/fstab 
```
> You can also delete it for regenerating a new one. And check the mounting operation anterior, use `umount` in case of errors and remount them.
```
# rm -rf /mnt/etc/fstab
```

### Change root into the new system
> After this step, our operations are equivalent to performing in the newly installed system. Installation media(USB flash drive) is able to be unmounted now.
```
# arch-chroot /mnt
```

### Time zone
1. Set the time zone
> Replace `/Europe/Madrid` with your `/Region/City`
```
# ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
```
2. Generate /etc/adjtime
```
# hwclock --systohc
```

### Localization
1. Edit `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` and other needed locales. 
```
# vim /etc/locale.gen
```
2. Generate the locales.
```
# locale-gen
```
3. Create and edit locale.conf. Input `LANG=en_US.UTF-8` to set the LANG variable.
```
# vim /etc/locale.conf
```
> If you set the console keyboard layout, make the changes persistent in vconsole.conf, for example
> `# /etc/vconsole.conf`
`KEYMAP=es`

### Network configuration
1. Create the hostname file, input a customized `myhostname` and save it.
```
# vim /etc/hostname
```
2. Edit `/etc/hosts` file.
```
# vim /etc/hosts
```
2. add the content to the end of the file. (replace `myhostname` with your own host name)
```
127.0.0.1	localhost
::1		    localhost
127.0.1.1	myhostname.localdomain	myhostname
```

### Initramfs*
Creating a new initramfs is usually not required, because mkinitcpio was run on installation of the kernel package with pacstrap.For LVM, system encryption or RAID, modify mkinitcpio.conf(5) and recreate the initramfs image:
```
# mkinitcpio -P
```

### Root password
Set the root password:
```
# passwd
```

### Add user and configrate sudo* 
> Replace `username` with your own user name
```
# useradd -m -G wheel username
```
```
# passwd username
```
> In order to use root operations under ordinary users, sudo cofiguration is needed.
```
# pacman -S sudo
```
```
# vim /etc/sudoers
```
> uncomment `# %wheel ALL=(ALL:ALL) ALL`.

### Install microcode 
Select one of these commands according to your device.
```
# pacman -S intel-ucode
# pacman -S amd-ucode
```

## Install packages
1. Update list of `pacman`
```
pacman -Syu
```
2. Install necessary packages
```
pacman -S dialog wpa_supplicant ntfs-3g networkmanager netctl git
```

## Boot loader
> with the example of `grub`, others Boot loader please check [Bootloader](https://wiki.archlinux.org/title/Arch_boot_process#Boot_loader) 
1. Reinstall linux kernel
```
# pacman -S linux
```
2. Install grub
```
# pacman -S os-prober ntfs-3g grub efibootmgr
```
3. Enable `os-prober`
> Uncomment `# GRUB_DISABLE_OS_PROBER=false`
```
# vim /etc/default/grub
```
4. Arrange grub
```
# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
```
5. Generate configuration file
```
grub-mkconfig -o /boot/grub/grub.cfg
```
6. Check if there exist option of Arch Linux in item `menuenrtry`
```
# vim /boot/grub/grub.cfg
```

## Create swap file
> Swap space can take the form of a disk partition or a file. Users may create a swap space during installation or at any later time as desired. Swap space can be used for two purposes, to extend the virtual memory beyond the installed physical memory (RAM), and also for suspend-to-disk support.
```
# dd if=/dev/zero of=/swapfile bs=1M count=8192 status=progress
```
> Replace `8192` with a necessary value (unit:Mb). Same as RAM of device in general way.
1. Change authority
```
# chmod 600 /swapfile
```
2. Set swap file
```
# mkswap /swapfile
```
3. Enable swap file
```
# swapon /swapfile
```
4. Set an access into `/etc/fstab` for swap file
> Add `/swapfile none swap defaults 0 0` in the end of the file with a new line.
```
# vim /etc/fstab
```

## Install Graphical user interface
Install xrog server
```
# pacman -S xorg
```
### KDE Plasma
1. Installation
```
# pacman -S plasma kde-applications
```
2. Install SDDM theme
```
# pacman -S sddm
```
3. Set SDDM boot at Start-up
```
# systemctl enable sddm
```

### GNOME
1. Installation
```
# pacman -S gnome gdm
```
2. Set GNOME boot at Start-up
```
# systemctl enable gdm
```

### Enable NetworkManager for Graphical user interface
```
# systemctl disable netctl
# systemctl enable NetworkManager
```

Reboot into the Graphical User Interface.
