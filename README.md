# Arch Linux Encrypted Dual Drive Installation Guide

This guide installs Arch Linux on a UEFI system with two encrypted drives:

- NVMe drive: Contains the system (EFI partition and an encrypted partition with LVM).
- SATA SSD: Used for media storage (encrypted as a single ext4 partition).

Both encrypted partitions are unlocked at boot using a single passphrase via systemd‑cryptsetup’s auto‑unlock (configured through kernel parameters). In addition, this guide covers detailed network setup (WiFi and Ethernet with MAC randomization), DNS with DNSSEC/DoT, and NTP configuration.

**Note:** All commands are executed as root (or with `sudo`). Adjust device names (e.g., `/dev/nvme0n1`, `/dev/sda`), locale, time zone, and network settings as needed. **Backup any important data before proceeding.**

---

## 1. Early Setup: Keyboard and Network (Live Environment)

### 1.1 Keyboard & WiFi Setup

```bash
# Load your keymap (example: br-abnt2)
loadkeys br-abnt2

# Unblock WiFi and bring up the interface
rfkill unblock wifi
ip link set wlan0 up

# Connect to wireless using iwctl
iwctl
station wlan0 scan
station wlan0 get-networks
station wlan0 connect YOUR_WIFI_SSID
exit

# Verify connectivity
ping -c3 gnu.org
```

## 2. Disk Partitioning and Preparation

### 2.1 Identify Your Disks

Use lsblk to confirm:

    NVMe drive (e.g., /dev/nvme0n1) for the system.
    SATA SSD (e.g., /dev/sda) for media storage.

### 2.2 Partition the Drives

```bash
# Wipe existing partition tables
sgdisk -Z /dev/nvme0n1
sgdisk -Z /dev/sda
```
#### Partition NVMe (System Disk)
```bash
gdisk /dev/nvme0n1
# Create two partitions:
# 1. EFI System Partition (ESP): 512 MB, type ef00, label "ESP"
# 2. Encrypted system partition: remainder of disk, type 8308, label "cryptlvm"
```
##### Partition SATA SSD (Media Disk)
```bash
gdisk /dev/sda
# Create one partition:
# 1. Encrypted media partition: full disk, type 8308, label "cryptmedia"
```
```bash
# Inform the kernel of changes
partprobe -s /dev/nvme0n1
partprobe -s /dev/sda
````

## 3. Encrypting the Partitions

Encrypt both the system and media partitions using LUKS (use the same strong passphrase):
```bash
# Encrypt system partition on NVMe
cryptsetup -s 512 -h sha512 -i 5000 luksFormat /dev/nvme0n1p2
cryptsetup luksOpen /dev/nvme0n1p2 cryptlvm

# Encrypt media partition on SATA SSD
cryptsetup -s 512 -h sha512 -i 5000 luksFormat /dev/sda1
cryptsetup luksOpen /dev/sda1 cryptmedia
```
Note: With proper kernel parameters (see Section 10), systemd‑cryptsetup caches your passphrase so you only need to enter it once at boot.

## 4. LVM Setup and Filesystem Creation (System Drive)

### 4.1 Create LVM on the Decrypted System Partition

```bash
pvcreate /dev/mapper/cryptlvm
vgcreate vg /dev/mapper/cryptlvm
lvcreate -L <xG> vg --name swap    # Recommended: 1.5× your RAM, where x is the swap size
lvcreate -l +100%FREE vg --name root
````

### 4.2 Format Partitions

```bash
# Format the EFI partition (NVMe partition 1)
mkfs.fat -F32 -n ESP /dev/nvme0n1p1

# Format the root LV
mkfs.ext4 -L ROOT /dev/vg/root

# Format the media partition on SATA SSD
mkfs.ext4 -L MEDIA /dev/mapper/cryptmedia

# Setup swap on the LV
mkswap -L SWAP /dev/vg/swap
```

## 5. Mounting Filesystems

Mount the partitions in the correct order:
```bash
# Mount the root filesystem
mount /dev/vg/root /mnt

# Create mountpoints for EFI and media
mkdir -p /mnt/efi /mnt/media

# Mount the EFI partition
mount /dev/nvme0n1p1 /mnt/efi

# Mount the media partition
mount /dev/mapper/cryptmedia /mnt/media

# Enable swap
swapon /dev/vg/swap
```

## 6. Base System Installation and fstab Generation

### 6.1 Install the Base System

Update mirrors and install packages with pacstrap:
```bash
reflector -f 5 -a 24 -c BR -p https --save /etc/pacman.d/mirrorlist --verbose

pacstrap -K /mnt base base-devel linux-zen linux-firmware amd-ucode cryptsetup lvm2 vim git iwd
# (For Intel CPUs, use intel-ucode instead of amd-ucode.)
```

### 6.2 Generate and Edit fstab

Generate fstab:
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```
Now, open vim to edit /mnt/etc/fstab and adjust the entries. For example, replace the generated entries with:
```bash
# EFI partition
UUID=<EFI_UUID>   /efi   vfat   defaults,noatime,fmask=0137,dmask=0027  0 2

# Root partition
UUID=<ROOT_UUID>  /      ext4   defaults,noatime   0 1

# Media partition
UUID=<MEDIA_UUID> /media ext4   defaults,noatime   0 2

# Swap
UUID=<SWAP_UUID>  none   swap   sw    0 0
```
Replace <EFI_UUID>, <ROOT_UUID>, <MEDIA_UUID>, and <SWAP_UUID> with the actual values from your system.

## 7. Post-pacstrap Configuration (Chroot)

Chroot into the new system:
```bash
arch-chroot /mnt bash
```

### 7.1 System Configuration
#### Clock and Timezone
```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```
#### Locale

Open vim to edit /etc/locale.gen and uncomment your desired locale (e.g., en_US.UTF-8 UTF-8):
```bash
vim /etc/locale.gen
```
Then run:
```bash
locale-gen
```
Now, create or edit /etc/locale.conf with vim:
```bash
vim /etc/locale.conf
```
Insert:
```bash
LANG=en_US.UTF-8
LC_COLLATE=C
```
#### Console Keymap

Edit /etc/vconsole.conf with vim:
```bash
vim /etc/vconsole.conf
```
Insert:
```bash
KEYMAP=br-abnt2
```
#### Hostname and Hosts

Set the hostname:
```bash
echo "arch" > /etc/hostname
```
Edit /etc/hosts with vim:
```bash
vim /etc/hosts
```
Insert:
```bash
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch.localdomain arch
```
#### User Setup

Create a new user and set passwords:
```bash
useradd -mG wheel yourusername
passwd yourusername   # Set user password
passwd                # Set root password
```
Enable sudo for the wheel group by editing the sudoers file:
```bash
EDITOR=vim visudo
```
Uncomment the line:
```bash
%wheel ALL=(ALL:ALL) ALL
```

## 8. Network Configuration

### 8.1 Enable Network Services

Enable the required network services:
```bash
systemctl enable systemd-networkd systemd-resolved systemd-timesyncd iwd
```

### 8.2 WiFi Configuration

Edit /etc/iwd/main.conf with vim:
```bash
vim /etc/iwd/main.conf
```
Insert:
```bash
[General]
use_default_interface=true
AddressRandomization=network
AddressRandomizationRange=full
```
Create the WiFi network configuration for systemd‑networkd:
```bash
vim /etc/systemd/network/wifi.network
```
Insert:
```bash
[Match]
Name=wlan0

[Network]
DHCP=yes
IPv6PrivacyExtensions=true
```

### 8.3 Ethernet Configuration

Create the Ethernet network configuration:
```bash
vim /etc/systemd/network/20-wired.network
```
Insert:
```bash
[Match]
Name=eth0

[Link]
RequiredForOnline=routable

[Network]
DHCP=yes
IPv6PrivacyExtensions=true
```
#### MAC Address Randomization for Ethernet

Create the MAC randomization file:
```bash
vim /etc/systemd/network/01-mac.link
```
Insert:
```bash
[Match]
PermanentMACAddress=xx:xx:xx:xx:xx:xx

[Link]
MACAddress=random
```
Also, create a udev rule to randomize MAC addresses. Edit:
```bash
vim /etc/udev/rules.d/81-mac-spoof.rules
```
Insert:
```bash
ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="xx:xx:xx:xx:xx:xx", RUN+="/usr/bin/ip link set dev $name address random"
```

### 8.4 DNS Configuration (systemd-resolved)

Edit /etc/systemd/resolved.conf with vim:
```bash
vim /etc/systemd/resolved.conf
```
Insert:
```bash
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com 2606:4700:4700::1111#cloudflare-dns.com 2606:4700:4700::1001#cloudflare-dns.com
FallbackDNS=9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
DNSSEC=yes
DNSOverTLS=yes
MulticastDNS=no
````

### 8.5 NTP Configuration (systemd-timesyncd)

Edit /etc/systemd/timesyncd.conf with vim:
```bash
vim /etc/systemd/timesyncd.conf
```
Insert:
```bash
[Time]
NTP=0.br.pool.ntp.org 1.br.pool.ntp.org 2.br.pool.ntp.org 3.br.pool.ntp.org
FallbackNTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
```

## 9. Configuring mkinitcpio and UKI

Before regenerating the initramfs, perform the following steps:

    Create the UKI directory:
```bash
mkdir -p /efi/EFI/Linux
```
Edit the preset file:
Open vim to edit /etc/mkinitcpio.d/linux-zen.preset:
```bash
vim /etc/mkinitcpio.d/linux-zen.preset
```
    Uncomment all lines related to UKI.
    Comment out all lines related to "image".
    Optional: Remove any mentions to initramfs from /boot if desired.

Edit /etc/mkinitcpio.conf with vim to set up the initramfs with systemd‑based hooks:
```bash
vim /etc/mkinitcpio.conf
```
Locate the line starting with HOOKS= and change it to:
```bash
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
```
You may remove fsck since it is handled by the systemd hook, same with block, but that is less recomended

## 10. Kernel Command Line Configuration (Unified Unlock)

Edit your kernel command line (for example, in /etc/kernel/cmdline or via your bootloader entry) to include:
```bash
rd.luks.name=<NVMe_UUID>=cryptlvm rd.luks.options=<NVMe_UUID>=discard \
rd.luks.name=<SATA_UUID>=cryptmedia rd.luks.options=<SATA_UUID>=discard \
root=UUID=<ROOT_UUID> resume=UUID=<SWAP_UUID> rw quiet [additional parameters]
```
    Important: Replace <NVMe_UUID>, <SATA_UUID>, <ROOT_UUID>, and <SWAP_UUID> with the actual UUIDs (e.g., obtained using blkid -s UUID -o value /dev/nvme0n1p2).

Now, regenerate the initramfs:
```bash
mkinitcpio -P
````

## 11. Boot Loader Installation (systemd‑boot with UKI)

Install systemd‑boot into the EFI partition:
```bash
bootctl install --esp-path=/efi
```
Ensure your loader configuration (for example, /boot/loader/loader.conf) and kernel entry files are set up so that the Unified Kernel Image (UKI) is detected automatically.

    Tip: Re-run bootctl install after updating fstab and your UKIs to refresh bootloader entries.

You may create a pacman hook to update bootctl whenever systemd is upgraded, however you should still pay attention to any changes that may warrent an update:
```bash
mkdir /etc/pacman.d/hooks
vim /etc/pacman.d/hooks/95-systemd-boot.hook
```
This hook will run anytime a systemd related package is upgraded.
```bash
/etc/pacman.d/hooks/95-systemd-boot.hook

[Trigger]
Type = Package
Operation = Upgrade
Target = systemd

[Action]
Description = Gracefully upgrading systemd-boot...
When = PostTransaction
Exec = /usr/bin/systemctl restart systemd-boot-update.service
```

## 12. Finalizing and Reboot

Exit the chroot and finish the installation:
```bash
exit
sync
poweroff
```
After reboot, you should be prompted for the encrypted volumes. Enter your passphrase once, and systemd‑cryptsetup will unlock both the system (LVM root) and media partitions automatically.
