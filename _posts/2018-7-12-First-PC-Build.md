---
layout: post
title: First PC Build - Installing Arch Linux
---

After getting the physical parts of my first computer together with the help of [pcpartpicker.com](pcpartpicker.com), it was time to install an OS.

From the start, I knew I wanted to install Linux, but I spent a lot of time picking a distro. It's easy enough to switch over (depending on the distros in question, of course). Still, I wanted to make an informed choice. A big motivation for making my own desktop from the ground up was the ability to customize every part. Ubuntu and Manjaro were high on my list especially since they are generally more newbie-friendly installations. Desktop environments and window managers are set right away and many apps are bundled in, too. In the end I chose [Arch](https://wiki.archlinux.org/index.php/Arch_compared_to_other_distributions). Unlike other distros, Arch requires getting your hands dirty, forcing you to interact with a shell from startup. If you don't know what you need and are afraid to poke around and learn about the nitty gritty of your computer, it's not a good time.

Despite the downsides for a new Linux user, I didn't mind a bit of extra work to get a fully customized machine. Furthermore, while I still have a stable, dependable MacBook Pro, it made sense to let myself tinker on the new machine as well.

Once I made my decision, it was time to get started. The best way to install is to follow the most current directions on the [Arch Wiki](https://wiki.archlinux.org/index.php/Installation_guide), but I kept my own notes as I went along.

## Pre-Installation

### Make a Bootable USB

I had a 16GB USB drive available, so I plugged it into my MacBook and ran

```
diskutil list
```

After looking at the list of disks available, I found the volume (`/dev/disk<number>`) corresponding to my thumb drive. I then unmounted the USB using

```
diskutil unmountDisk /dev/disk2
```

Then I downloaded the Arch .iso from a mirror and moved it to my USB with `dd`.

```
sudo dd if=archlinux-2018.07.01-x86_64.iso of=/dev/rdisk2 bs=1m
```

I waited until I could see that my laptop was unable to recognize the USB disk.

## Booting Up

First, I verified that the boot mode of my motherboard was UEFI.
```
ls /sys/firmware/efi/efivars
```
I had tried to verify the boot mode a few times. It seemed like I could chose the boot mode from the ASUS BIOS screen. When I explicitly picked UEFI from my USB live iso, this command ran successfully.


## Set up the Network

I had a lot of trouble connecting with Ethernet (both on my new desktop and on my prior trials with a MacBook). Once I had a wireless network PCI card in my tower, I was able to install over WiFi using a dialog from

```
wifi-menu
```

I used the free wifi that did not require sign in because I wanted to wait until later to deal with it (once I was all installed and set up).

## Partitioning

Since I had UEFI enabled, I used GPT to create an EFI system partition (ESP). An EFI system partition on GPT is identified by the partition type GUID `C12A7328-F81F-11D2-BA4B-00A0C93EC93B`. The Arch wiki recommends GPT because some UEFI firmwares do not allow UEFI/MBR boot.

### Create the Partitions

I used `fdisk` to create a partition with partition type `EFI system`.

The following layout was my choice for a partition scheme.

| **Mount Point** | **Partition** | **Partition Type GUID** | **Size** |
|-------------|-----------|---------------------|------|
|`/boot`|`/dev/sda1`| EFI system partition `C12A7328-F81F-11D2-BA4B-00A0C93EC93B`| 550MiB |
|`/`    |`/dev/sda2`| Linux x86-64 root (/) `4F68BCE3-E8CD-4DB1-96E7-FBCAF984B709` | 25GiB |
|`[SWAP]`|	`/dev/sda3` |	Linux swap `0657FD6D-A4AB-43C4-84E5-0933C84B4F4F` 		| 1GiB |
|`/home` |	`/dev/sda4` |	Linux /home `933AC7E1-2EB4-4F13-B844-0E14E2AEF915` 	|	Remainder of the device (905GiB) |

I checked that everything looked good using

```
fdisk -l
```

### Formatting the Partitions

**EFI System**: (You *must* use FAT32 formatting for the EFI partition.)
```
mkfs.fat -F32 /dev/sda1
```
**Root and /home**:
```
mkfs.ext4 /dev/sda2
mkfs.ext4 /dev/sda4
```

**Swap**: (You need to initialize when you make a swap partition.)

```
mkswap /dev/sda3
swapon /dev/sda3
```

*Note*: To overwrite old file systems on a partition, use `mkfs.ext4 /dev/sda2` to overwrite. For large partitions, it may take a minute to discard device blocks.

### Mount the Partitions

The Arch wiki says:

> The kernels and initramfs files need to be accessible by the boot loader or UEFI itself to successfully boot the system. Thus if you want to keep the setup simple, your boot loader choice limits the available mount points for EFI system partition.

For GRUB (my bootloader of choice), and UEFI, I used the following set up.

```
mount /dev/sda2 /mnt
mkdir /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
mkdir /mnt/home
mount /dev/sda4 /mnt/home
```
## Install the Base Packages

```
pacstrap /mnt base
```

## Configuring the System

Generate an `fstab ` file:

```
genfstab -U /mnt >> /mnt/etc/fstab
```

Check the file using `cat /mnt/etc/fstab`.

### Change Root into New System (`chroot`)

```
arch-chroot /mnt
```
I knew I needed `dialog`, `wpa_supplicant`, and `iw` if I wanted to continue using WiFi for networking after moving off the live USB stick, so I downloaded these packages from chroot.
```
pacman -S dialog iw wpa_supplicant
```

### Time-Zone

Exit from `chroot` then:

```
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
hwclock --systohc
```

### Locale

Uncomment en_US.UTF-8 UTF-8 from locales:

```
vi /etc/locale.gen
```

Generate by:

```
locale-gen
```

Set the `LANG` variable in `locale.conf` as `LANG=en_US.UTF-8`:

```
vi /etc/locale.conf
```


### Network Configuration

Create a hostname file and include just the hostname. I chose to make my hostname "rhubarb".

```
vi /etc/hostname
```

Add hosts to `/etc/hosts`.

```
127.0.0.1	localhost
::1		localhost
127.0.1.1	rhubarb.localdomain	rhubarb
```

Set the root password:
```
passwd
```

### Install GRUB

Then I installed my bootloader.

```
pacman -S grub efibootmgr intel-ucode
```

I made sure I was in chroot, then:

```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig - o /boot/grub/grub.cfg
```

## Reboot

Exit chroot, then unmount all partitions:

```
umount -R /mnt
```

Restart and remove USB stick:

```
systemctl reboot
```

---
After this, my OS was installed and ready to be customized!
