Current as of November 2022, A brief install guide for Archlinux on a Zephyrus G14 laptop (AMD Ryzen 4800HS + Nvidia 1650Ti). Should apply to other devices. Sets up a very basic (and fast) KDE desktop enviroment.

Assumptions: 
* NVME SSD installed in the first slot of your motherboard
* EFI system
* You use wifi, and your wifi module/USB stick/etc. is supported. For ethernet only, ignore the first step as it's not needed.
* Root partition is not encrypted. I don't care about encryption, there are guides elsewhere for this.
* This guide is for AMD processors - no real difference for Intel, replace amd-ucode with intel-ucode later on

# basics
Connect to wifi, skip if on an ethernet connection

    iwctl
    station wlan0 connect <SSID>
    exit

Sync time

    timedatectl set-ntp true

# filesystem
Partition the disk

    cfdisk /dev/nvme0n1

Create a 512M partition for EFI (type EFI)
Create the rest as a linux partition (we'll use ext4)

Format the partitions, EFI uses FAT32, remainder ext4

    mkfs.fat -F32 /dev/nvme0n1p1
    mkfs.ext4 /dev/nvme0n1p2


Mount the root partition, create a boot directory and mount the EFI partition

    mount /dev/nvme0n1p2 /mnt
    mkdir -p /mnt/boot
    mkdir -p /mnt/boot/efi
    mount /dev/nvme0n1p1 /mnt/boot/efi


# base system

    pacstrap /mnt base base-devel linux linux-firmware nano 

    genfstab -U /mnt >> /mnt/etc/fstab

# chroot

    arch-chroot /mnt

# set timezone

    ln -sf /usr/share/zoneinfo/Pacific/Auckland /etc/localtime

    hwclock --systohc

# locale

    nano /etc/locale.gen (uncomment en_US.UTF-8 etc.)
    locale-gen
    echo "LANG=en_US.UTF-8" > /etc/locale.conf

# hostname

    echo "archlaptop" > /etc/hostname
    echo "127.0.0.1 localhost
    ::1 localhost
    127.0.0.1 archlaptop.localdomain archlaptop" >> /etc/hosts

# password

    passwd
    <MY ROOT PASSWORD GOES HERE>

    useradd -m <user>
    usermod -aG wheel,audio,video,optical,storage <user>
    passwd <user>
    nano /etc/sudoers (enable the wheel group)

# swapfile
Creates a 16GB swapfile

    dd if=/dev/zero of=/swapfile bs=1M count=16384 status=progress
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile

    nano /etc/fstab (add following:)
    /swapfile none swap defaults 0 0

# bootloader
Install bootloader, plus the networking stuff we'll need after a reboot

    pacman -S grub efibootmgr networkmanager wpa_supplicant amd-ucode

    grub-install /dev/nvme0n1
    grub-mkconfig -o /boot/grub/grub.cfg

    systemctl enable NetworkManager
    systemctl enable wpa_supplicant

Fix slow Intel AX200 wifi:

    echo "options iwlwifi bt_coex_active=0 swcrypto=1" > /etc/modprobe.d/iwlwifi.conf

Reboot

    exit
    reboot

# connect

    nmcli device wifi connect <SSID> password <SSID_PASSWORD>

# basics
Update package manager

    pacman -Syu
    
Install a reasonably minimal KDE plasma desktop plus a set of basic utils and drivers for the NVIDIA card and bluetooth

    pacman -S xorg-server xf86-video-amdgpu mesa mesa-demos firefox plasma-desktop dolphin konsole kate pipewire pipewire-pulse pipewire-alsa bluez bluez-utils sddm nvidia nvidia-utils nvidia-prime powerdevil bluedevil nvtop plasma-nm bash-completion breeze-gtk kde-gtk-config ark p7zip unrar git

    systemctl enable sddm
    systemctl enable bluetooth

    reboot

# Virtual Surround Sound

    mkdir ~/.config/pipewire
    mkdir ~/.config/pipewire/filter-chain.conf.d

    cp /usr/share/pipewire/filter-chain/sink-virtual-surround-7.1-hesuvi.conf ~/.config/pipewire/filter-chain.conf.d/

Download a HRTF file from https://airtable.com/shruimhjdSakUPg2m/tbloLjoZKWJDnLtTc (the HeSuVi wave files, you'll want 48kHz versions where possible)

Edit the .conf file, replace the hesuvi_hrir/hrir.wav occurances with the full path to your model file

Test with

    pipewire -c filter-chain.conf

A 'Virtual Surround Sink' device should be available. Test with a 5.1/7.1 sound/video file

To run automatically on startup

    cp /usr/share/pipewire/pipewire.conf ~/.config/pipewire/

Edit the .conf file to change the context.exec section to:

    context.exec = [{ path = "/usr/bin/pipewire" args = "-c filter-chain.conf" }]
