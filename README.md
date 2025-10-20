# Arch Linux Encrypted Dual Drive Installation Guide

This guide installs Arch Linux on a UEFI system with multiple encrypted drives:

- **NVMe drive**: `${SYS_DISK}` — contains the system (EFI partition and an encrypted partition with LVM).  
- **SATA SSD**: `${MEDIA_DISK}` — used for media storage (encrypted as a single ext4 partition).  
- **Optional**: Secure Boot and TPM 2.0.

If you plan to use Secure Boot, set it to **Setup Mode** in the BIOS. If using TPM, ensure your system supports **TPM 2.0** since this guide does not cover older versions. You can skip Secure Boot or TPM steps if you do not need them, and you can choose to use only one or the other. However, if you use TPM without Secure Boot, remove `7` from `--tpm2-pcrs=0+7+15` in Section 14. If you only have one disk, ignore commands related to the second disk (`${MEDIA_DISK}`).

Both encrypted partitions are unlocked at boot using a single passphrase via `systemd-cryptsetup` auto-unlock (configured through kernel parameters, meaning no `crypttab`) and TPM. This guide also covers detailed network setup (Wi-Fi and/or Ethernet with MAC randomization, DNS with DNSSEC/DoT, and NTP/NTS).

**Note:** All commands assume root privileges (or `sudo`). Adjust device names (e.g., `/dev/nvme0n1`, `/dev/sda`), locale (e.g., `en_US.UTF-8`), timezone, and network settings (e.g., `wlan0`, `eth0`) as needed. **Backup important data before proceeding!**

---

## 1. Early Setup: Keyboard and Network (Live Environment)

### 1.1 Keyboard & Wi-Fi Setup

```bash
# Load your keymap (example: br-abnt2)
loadkeys br-abnt2

# Unblock Wi-Fi and bring up the interface
rfkill unblock wifi
ip link set wlan0 up

# Connect to Wi-Fi using iwctl
iwctl
station wlan0 scan
station wlan0 get-networks
station wlan0 connect YOUR_WIFI_SSID
exit

# Verify connectivity
ping -c 3 gnu.org
```

---

## 2. Disk Partitioning and Preparation

### 2.1 Identify your disks

Use `lsblk` to confirm disk names:

- NVMe drive (e.g., `/dev/nvme0n1`) for the system.  
- SATA SSD (e.g., `/dev/sda`) for media storage.

### 2.2 Export your drive paths

```bash
# Example:
# export SYS_DISK="/dev/nvme0n1"
# export MEDIA_DISK="/dev/sda"

# In a VM, your main disk might be /dev/vda and your second /dev/vdb.
# Always check with lsblk.

export SYS_DISK="/dev/nvme0n1"
export MEDIA_DISK="/dev/sda"

# If a disk is NVMe, then values like ${SYS_DISK}2 or ${SYS_DISK}1 should instead be ${SYS_DISK}p2 or ${SYS_DISK}p1
```

### 2.3 Partition the drives

```bash
# Wipe existing partition tables
sgdisk -Z ${SYS_DISK}
sgdisk -Z ${MEDIA_DISK}
```

#### Partition NVMe (system disk)

```bash
gdisk ${SYS_DISK}
# Create two partitions:
# 1. EFI System Partition (ESP): 512 MB, type ef00, label "ESP"
# 2. Encrypted system partition: remainder of disk, type 8309, label "cryptlvm"
```

#### Partition SATA SSD (media disk)

```bash
gdisk ${MEDIA_DISK}
# Create one partition:
# 1. Encrypted media partition: full disk, type 8309, label "cryptmedia"
```

```bash
# Inform the kernel of partition changes
partprobe -s ${SYS_DISK}
partprobe -s ${MEDIA_DISK}
```

---

## 3. Encrypting the partitions

Encrypt both the system and media partitions using LUKS (use the same strong passphrase):

```bash
# Encrypt system partition on NVMe
cryptsetup luksFormat ${SYS_DISK}2
cryptsetup luksOpen ${SYS_DISK}2 cryptlvm

# Encrypt media partition on SATA SSD
cryptsetup luksFormat ${MEDIA_DISK}1
cryptsetup luksOpen ${MEDIA_DISK}1 cryptmedia
```

**Note:** With proper kernel parameters (see Section 10), `systemd-cryptsetup` can cache your passphrase.

---

## 4. LVM setup and filesystem creation (system drive)

### 4.1 Create LVM on the decrypted system partition

```bash
pvcreate /dev/mapper/cryptlvm
vgcreate vg /dev/mapper/cryptlvm

# Recommended: SWAP ≥ RAM for hibernation or at least 4G.
lvcreate -L <x>G vg --name swap  # Replace <x> with swap size
lvcreate -l +100%FREE vg --name root
```

### 4.2 Format partitions

```bash
# Format EFI partition
mkfs.fat -F32 -n ESP ${SYS_DISK}1

# Format root LV
mkfs.ext4 -L ROOT /dev/vg/root

# Format media partition
mkfs.ext4 -L MEDIA /dev/mapper/cryptmedia

# Setup swap
mkswap -L SWAP /dev/vg/swap
```

---

## 5. Mounting filesystems

```bash
# Mount root filesystem
mount /dev/vg/root /mnt

# Create mountpoints
mkdir -p /mnt/efi /mnt/media

# Mount EFI partition
mount ${SYS_DISK}1 /mnt/efi

# Mount media partition
mount /dev/mapper/cryptmedia /mnt/media

# Enable swap
swapon /dev/vg/swap
```

---

## 6. Base system installation and fstab

### 6.1 Install the base system

```bash
reflector -f 5 -a 24 -c BR -p https --save /etc/pacman.d/mirrorlist --verbose
# Add your country code after `-c`; here `BR` is used as an example.

pacstrap -K /mnt base base-devel linux-zen linux-firmware amd-ucode cryptsetup lvm2 vim git unbound expat efibootmgr iwd openssh sbctl chronyd
# Use `intel-ucode` for Intel CPUs instead of `amd-ucode`.
# Install `sbctl` only if you want Secure Boot.
# `openssh` is optional.
# `unbound` and `expat` are needed for DNS; `chronyd` is used for NTP/NTS.
```

### 6.2 Generate and edit fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Open `/mnt/etc/fstab` with `vim`:

```bash
vim /mnt/etc/fstab
```

Adjust `fmask` and `dmask` values from `0022` to `0137` and `0027`, respectively. Replace `relatime` with `noatime` if you prefer no access-time updates.

Example `fstab` entries (<placeholders> shown instead of actual UUIDs):

```ini
# /dev/mapper/vg-root LABEL=ROOT
UUID=<ROOT_UUID>   /       ext4    rw,noatime  0 1

# /dev/nvme0n1p1 LABEL=ESP
UUID=<ESP_UUID>    /efi    vfat    rw,noatime,fmask=0137,dmask=0027,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro  0 2

# /dev/mapper/cryptmedia LABEL=MEDIA
UUID=<MEDIA_UUID>  /media  ext4    rw,noatime  0 2

# /dev/mapper/vg-swap LABEL=SWAP
UUID=<SWAP_UUID>   none    swap    defaults    0 0
```

---

## 7. Post-pacstrap configuration (chroot)

Chroot into the new system:

```bash
arch-chroot /mnt bash
```

### 7.1 System configuration

#### Clock and timezone

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```

#### Locale

Open `/etc/locale.gen` with `vim` and uncomment your locale (e.g., `en_US.UTF-8 UTF-8`):

```bash
vim /etc/locale.gen
```

Generate locales:

```bash
locale-gen
```

Open `/etc/locale.conf` with `vim`:

```bash
vim /etc/locale.conf
```

Insert:

```ini
LANG="en_US.UTF-8"
LC_COLLATE="C"
```

(Using `export` in `/etc/locale.conf` is unnecessary, `/etc/locale.conf` typically contains variable assignments without `export`.)

#### Console keymap

Open `/etc/vconsole.conf` with `vim`:

```bash
vim /etc/vconsole.conf
```

Insert:

```ini
KEYMAP=br-abnt2
```

#### Hostname and hosts

Set hostname:

```bash
echo "arch" > /etc/hostname
```

Open `/etc/hosts` with `vim`:

```bash
vim /etc/hosts
```

Note: the first four lines are typically sufficient for a basic configuration.

```ini
127.0.0.1        localhost local
127.0.1.1        arch.localdomain arch
255.255.255.255  broadcasthost
::1              localhost ip6-localhost ip6-loopback
fe80::1%lo0      localhost
ff00::0          ip6-localnet ip6-mcastprefix
ff02::1          ip6-allnodes
ff02::2          ip6-allrouters
ff02::3          ip6-allhosts
0.0.0.0          0.0.0.0
```

#### User setup

Create a user and set passwords:

```bash
useradd -mG wheel $USER
passwd $USER          # set password for the newly created user
passwd                 # set root password
```

Enable `sudo` for the `wheel` group:

```bash
EDITOR=vim visudo
```

Uncomment:

```ini
%wheel ALL=(ALL:ALL) ALL
```

---

## 8. Network configuration

### 8.1 Enable network services

Enable the services you need (this also enables `fstrim.timer` for regular SSD trimming):

```bash
systemctl enable systemd-networkd unbound fstrim.timer iwd openssh chronyd
```

### 8.2 Wi-Fi: MAC randomization & DHCPv4 anonymization

Create the iwd configuration directory and file:

```bash
mkdir -p /etc/iwd
vim /etc/iwd/main.conf
```

Insert:

```ini
[General]
AddressRandomization=network
AddressRandomizationRange=full

[DriverQuirks]
PowerSaveDisable=*
```

For network-specific `.psk` files under `/var/lib/iwd/`, add at the end:

```ini
[Settings]
AlwaysRandomizeAddress=true
```

Create the systemd-networkd file for Wi-Fi:

```bash
vim /etc/systemd/network/wifi.network
```

Insert:

```ini
[Match]
Name=wlan0

[Network]
DHCP=yes
IPv6PrivacyExtensions=true

[DHCPv4]
Anonymize=true
```

Use `ip link` to check the actual interface name; `wlan0` is only an example.

### 8.3 Ethernet: MAC randomization & DHCPv4 anonymization

```bash
vim /etc/systemd/network/wired.network
```

Insert:

```ini
[Match]
Name=eth0

[Link]
RequiredForOnline=routable

[Network]
DHCP=yes
IPv6PrivacyExtensions=true

[DHCPv4]
Anonymize=true
```

Use `ip link` to check the actual interface name; `eth0` is only an example.

Create MAC randomization policy:

```bash
vim /etc/systemd/network/01-mac.link
```

Insert:

```ini
[Match]
MACAddressPolicy=random
```

### 8.4 Resolve DNS with Unbound and AdGuard

Fetch the `unbound.conf` from the referenced repo:

```bash
git clone https://codeberg.org/Unclaimed3646/unbound.git
cd unbound
cat unbound.conf > /etc/unbound/unbound.conf
```

### 8.5 Configure chronyd as an NTP/NTS client

```bash
vim /etc/chrony.conf
```

Insert:

```ini
server gps.ntp.br iburst nts
```

You will want to change the server to one close to you; see https://github.com/jauderho/nts-servers for NTS-capable servers.

---

## 9. Configuring mkinitcpio and UKI

```bash
mkdir -p /efi/EFI/Linux
vim /etc/mkinitcpio.d/linux-zen.preset
```

- Uncomment all lines related to UKI.  
- Comment out `image` lines if you are using UKI only.  
- (Optional) Remove `/boot` initramfs if desired.

Open `/etc/mkinitcpio.conf` with `vim`:

```bash
vim /etc/mkinitcpio.conf
```

Set the `HOOKS=` line (example):

```ini
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
```

---

## 10. Kernel command line (unified unlock)

Open `/etc/kernel/cmdline` and insert something like:

```bash
rd.luks.name=${SYS_DISK}2=cryptlvm rd.luks.options=${SYS_DISK}2=timeout=300s,discard,x-initrd.attach,password-echo=no,tries=5,tpm2-measure-pcr=yes \
rd.luks.name=${MEDIA_DISK}1=cryptmedia rd.luks.options=${MEDIA_DISK}1=timeout=300s,discard,x-initrd.attach,password-echo=no,tries=5,tpm2-measure-pcr=yes \
root=UUID=<ROOT_UUID> resume=UUID=<SWAP_UUID> rw [additional_parameters]
```

Note: `x-initrd.attach` and `tpm2-measure-pcr=yes` is only needed if you are using TPM. Replace `${SYS_DISK}2`, `${MEDIA_DISK}1`, `<ROOT_UUID>`, and `<SWAP_UUID>` with the actual device names and UUIDs (for example, obtained using `blkid -s UUID -o value ${SYS_DISK}2` and `blkid -s UUID -o value /dev/mapper/vg-root`).

Example additional kernel parameters:

```bash
# Example additional parameters:
quiet mitigations=off amdgpu.dc=1 loglevel=3 systemd.show_status=auto rd.udev.log_level=3 zswap.compressor=lz4 sysctl.vm.swappiness=10 nowatchdog module_blacklist=nouveau,i915,radeon,iTCO_wdt,sp5100_tco,wdat_wdt,pcspkr,mgag200,uvcvideo
```

Then regenerate initramfs:

```bash
mkinitcpio -P
```

---

## 11. EFISTUB

Create an EFISTUB entry:

```bash
efibootmgr -d ${SYS_DISK} -p 1 -c -L Arch -l '\EFI\Linux\arch-linux-zen.efi' -v
```

---

## 12. Enabling Secure Boot

Check sbctl status and verify:

```bash
sbctl status
sbctl verify
```

Create and enroll keys:

```bash
sbctl create-keys
sbctl enroll-keys -m
```

Sign UKIs:

```bash
sbctl sign -s /efi/EFI/Linux/arch-linux-zen.efi
sbctl sign -s /efi/EFI/Linux/arch-linux-zen-fallback.efi
```

---

## 13. Reboot to save status before enabling TPM

```bash
exit
sync
systemctl reboot --firmware-setup
```

Before booting, re-enable Secure Boot in your BIOS to its regular state if you plan on using it.

Reboot. At boot, enter the passphrase once; `systemd-cryptsetup` will unlock both partitions automatically (if configured).

---

## 14. Enabling TPM

Export your drive paths again (repeat if needed):

```bash
# Example:
# export SYS_DISK="/dev/nvme0n1"
# export MEDIA_DISK="/dev/sda"

# In a VM, your main disk might be /dev/vda and your second /dev/vdb.
# Always check with lsblk.

export SYS_DISK="/dev/nvme0n1"
export MEDIA_DISK="/dev/sda"

# If a disk is NVMe, then values like ${SYS_DISK}2 or ${SYS_DISK}1 should instead be ${SYS_DISK}p2 or ${SYS_DISK}p1
```

Create a recovery key (optional):

```bash
systemd-cryptenroll ${SYS_DISK}2 --recovery-key
systemd-cryptenroll ${MEDIA_DISK}1 --recovery-key
```

Enroll TPM (remove `7` from `--tpm2-pcrs=0+7+15` if you are not using Secure Boot). There are 64 zeros in the example below:

```bash
systemd-cryptenroll --wipe-slot tpm2 --tpm2-device=auto --tpm2-pcrs=0+7+15:sha256=0000000000000000000000000000000000000000000000000000000000000000 --tpm2-with-pin=yes ${SYS_DISK}2
systemd-cryptenroll --wipe-slot tpm2 --tpm2-device=auto --tpm2-pcrs=0+7+15:sha256=0000000000000000000000000000000000000000000000000000000000000000 --tpm2-with-pin=yes ${MEDIA_DISK}1
```

Add TPM module support (list TPM devices and add module to mkinitcpio.conf ):

```bash
systemd-cryptenroll --tpm2-device=list
vim /etc/mkinitcpio.conf
```

Add the appropriate module name(s) to `MODULES=` array.

```bash
mkinitcpio -P
```

---

## 15. Point `resolv.conf` to loopback

```bash
vim /etc/resolv.conf
```

Insert:

```ini
nameserver ::1
nameserver 127.0.0.1
options edns0 trust-ad
```

Prevent future overwriting:

```bash
sudo chattr +i /etc/resolv.conf
```

---

# Post-install addons

From now on, commands show `sudo` when appropriate.

### 1. Silent boot autologin

```bash
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d/
sudo vim /etc/systemd/system/getty@tty1.service.d/autologin.conf
```

Insert:

```ini
[Service]
ExecStart=
ExecStart=-/usr/bin/agetty --skip-login --nonewline --noissue --autologin $USER --noclear %I $TERM
```

### 2. Enable sysrq

```bash
echo "kernel.sysrq = 1" | sudo tee /usr/lib/sysctl.d/50-sysrq.conf >/dev/null
```

### 3. PipeWire & WirePlumber

Install PipeWire and WirePlumber:

```bash
sudo pacman -S pipewire pipewire-pulse pipewire-alsa pipewire-jack wireplumber
```

Enable the user services (do this as the user, not as root):

```bash
systemctl --user enable pipewire pipewire-pulse wireplumber
```

### 4. Media subdirectories access

Create a top-level media directory, then create a subdirectory and set permissions (example):

```bash
sudo mkdir -p /media
cd /media
sudo mkdir foo
sudo chown root:wheel foo
sudo chmod 770 foo
ln -s /media/ ~/bar
```

### 5. GPG key issues

```bash
gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv X
```

### 6. Paru (AUR helper)

Do not run `makepkg` as `root` or with `sudo`. Install build tools and paru:

```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru-bin.git
cd paru
makepkg -si
```

### 7. ALHP optimized repo

Check if your CPU supports the optimizations:

```bash
/lib/ld-linux-x86-64.so.2 --help
```

Use all optimization levels your CPU supports (e.g., `x86-64-v3`, `x86-64-v2`) so the system prefers v3 but can fall back.

Get keys and enable the repos:

```bash
paru -S alhp-keyring alhp-mirrorlist
```

Modify `/etc/pacman.conf` to include the new mirrorlists, for example:

```ini
[core-x86-64-v3]
Include = /etc/pacman.d/alhp-mirrorlist

[core-x86-64-v2]
Include = /etc/pacman.d/alhp-mirrorlist

[core]
Include = /etc/pacman.d/mirrorlist

[extra-x86-64-v3]
Include = /etc/pacman.d/alhp-mirrorlist

[extra-x86-64-v2]
Include = /etc/pacman.d/alhp-mirrorlist

[extra]
Include = /etc/pacman.d/mirrorlist

# if you need [multilib] support
[multilib-x86-64-v3]
Include = /etc/pacman.d/alhp-mirrorlist

[multilib-x86-64-v2]
Include = /etc/pacman.d/alhp-mirrorlist

[multilib]
Include = /etc/pacman.d/mirrorlist
```

Update your packages (do not run `paru` as root):

```bash
paru
```

### 8. Chrooting into an encrypted system with LVM if needed

```bash
sudo cryptsetup luksOpen /dev/nvme0n1p2 cryptlvm
sudo vgchange -ay
sudo mount /dev/mapper/vg-root /mnt
sudo arch-chroot /mnt bash
```

### 9. Disabling sleep key (acpi_event)

If your sleep key is problematic (e.g., too close to `Esc`), add to `/etc/systemd/logind.conf`:

```ini
HandleSuspendKey=ignore
```

### 10. Using TLP

```bash
sudo pacman -S tlp
sudo systemctl enable tlp
sudo systemctl mask systemd-rfkill.service systemd-rfkill.socket
```

Edit TLP settings:

```bash
sudo vim /etc/tlp.d/10-wifi-bluetooth.conf
```

Insert:

```ini
DEVICES_TO_ENABLE_ON_STARTUP="wifi bluetooth"
```

```bash
sudo vim /etc/tlp.d/20-disable-usb-autosuspend.conf
```

Insert:

```ini
USB_AUTOSUSPEND=0
```

---
**TODO:** dm-crypt headerless option, script version.
