# Artix Installation Guide by Me

Installation guide for Artix Openrc with logical volume manager. This guide use UEFI and GPT partition table.

## Table of content
- [1. Create partition](#1-create-partition)
- [2. Format partition](#2-format-partition)
- [3. Create physical volume, volume group, logical volume](#3-create-physical-volume-volume-group-logical-volume)
- [4. Format logical volume](#4-format-logical-volume)
- [5. Mount EFI and logical volume](#5-mount-efi-and-logical-volume)
- [6. Install Artix](#6-install-artix)
- [7. Configure Artix](#7-configure-artix)



### 1. Create partition
- - -
- Create partition with cfdisk, replace X with your disk name.

    ```
    cfdisk /dev/sdX
    ```

- Choose gpt type.
- Create partition 1 with 100M size, change type to "EFI Filesystem".
- Create partition 2 with remaining space, change type to "Linux filesystem".
- Write changes and exit.

### 2. Format partition
- - -
- Format partition 1 EFI Filesystem.

    ```
    mkfs.vfat -F32 /dev/sdX1
    ```

- Add label to EFI partition. (optional)

    ```
    fatlabel /dev/sdX1 "EFI"
    ```

### 3. Create physical volume, volume group, logical volume
- - -
- Create physical volume with pvcreate.

    ```
    pvcreate /dev/sdX2
    ```

- Create volume group with vgcreate. Replace "vgname" with your volume group name you want to create. 

    ```
    vgcreate vgname /dev/sdX2
    ```

- Create logical volume with lvcreate. Replace "lvname" with your logical volume name you want to create.

    ```
    lvcreate -l 100%FREE -n lvname vgname
    ```

### 4. Format logical volume
- - -
- Format logical volume with mkfs.ext4 if you want to use as ext4 filesystem.

    ```
    mkfs.ext4 /dev/vgname/lvname -L ROOT
    ```

- Format logical volume with mkfs.xfs if you want to use as xfs filesystem.

    ```
    mkfs.xfs /dev/vgname/lvname -L ROOT
    ```

- Format logical volume with mkfs.btrfs if you want to use as btrfs filesystem.
    ```
    mkfs.btrfs /dev/vgname/lvname -L ROOT
    ```

### 5. Mount EFI and logical volume
- - -
- Mount logical volume with mount.

    ```
    mount /dev/vgname/lvname /mnt
    ```
- Create boot directory.

    ```
    mkdir -p /mnt/boot/efi
    ```
- Mount EFI partition with mount.

    ```
    mount /dev/sdX1 /mnt/boot/efi
    ```

### 6. Install Artix
- - -
- Install Artix with basestrap.

    ```
    basestrap /mnt base base-devel openrc elogind-openrc
    ```

- Install Linux kernel and Linux firmware. If you want to use lts kernel, use "linux-lts" instead of "linux".

    ```
    basestrap /mnt linux linux-firmware
    ```

### 7. Configure Artix
- - -
- Chroot to /mnt and configure Artix.

    ```
    artix-chroot /mnt
    ```

- Set system clock. Replace region with your region and city too.

    ```
    ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
    ```

- Run hwclock to generate /etc/adjtime

    ```
    hwclock --systohc
    ```

- Localization

    Install a text editor of your choice (let's use nano here) and edit /etc/locale.gen, uncommenting the locales you desire

    ```
    pacman -S nano
    nano /etc/locale.gen
    ```
    Generate your desired locales running
    
    ```
    locale-gen
    ```

- Add lvm2 and resume modules to mkinitcpio
    ```
    nano /etc/mkinitcpio.conf
    ```
    insert "lvm2" and "resume" options manually to the following places between block and filesystems

    > HOOKS="base udev autodetect modconf block keyboard keymap lvm2 resume filesystems fsck"

- Install dependencies
    
    ```
    pacman -S device-mapper-openrc lvm2-openrc grub os-prober efibootmgr openssh openssh-openrc dhclient
    ```
- Install grub

    ```
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub
    ```
- Generate grub configuration

    ```
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
- Activate services
    
    ```
    rc-update add device-mapper boot
    rc-update add lvm boot
    rc-update add sshd default
    ```
- Set the root passwd
    
    ```
    passwd
    ```
- Enable ssh root login

    ```
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
    ```
- Set the hostname and hosts
    ```
    echo 'yourhostname' > /etc/hostname
    echo 'hostname="yourhostname"' > /etc/conf.d/hostname
    echo '127.0.0.1 localhost' >> /etc/hosts
    echo '::1 localhost' >> /etc/hosts
    echo '127.0.1.1 yourhostname.localdomain  yourhostname' >> /etc/hosts
    ```
- Set networking

    Get the exact name of your interface
    ```
    ip -s link
    ```
    Add config to /etc/conf.d/net | replace "your-interface" name with your real interface name

    ```
    echo 'config_your-interface=( "dhcp" )' >> /etc/conf.d/net
    ```
    Create symlink to openrc init
    ```
    ln -s /etc/init.d/net.lo /etc/init.d/net.your-interface
    ```
    Activate networking services
    ```
    rc-update add net.your-interface default
    ```
- Exit chroot, add fstab, unmount and reboot if you want to use the new system.
    ```
    exit
    fstabgen -U /mnt >> /mnt/etc/fstab
    umount -R /mnt
    reboot
    ```

<div align="center">

- - -
#### [Back to top](#artix-installation-guide-by-me)


</div>