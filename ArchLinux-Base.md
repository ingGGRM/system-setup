# Arch Linux System Architecture: ingGGRM-ArchLinux

**Hardware Target:** Acer Nitro 5 (i5-10300H, 16GB RAM, GTX 1650, 2x 500GB NVME)
**Design Philosophy:** Minimalist, high-performance, manual control, aesthetic flexibility.

---

## 1. Disk Partitioning GPT Table Layout Creation *Thinking in the rest of the disk 1 and the whole disk 2 as a single LVM partition.

### Disk 1
```bash
# 1. Create 1GB EFI Partition (Type EF00)
sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"EFI" /dev/nvme0n1
# 2. Create 150GB BTRFS Root Partition (Type 8300)
sgdisk -n 2:0:+150G -t 2:8300 -c 2:"Arch_Root" /dev/nvme0n1
# 3. Create LVM Partition with remaining space (Type 8E00)
sgdisk -n 3:0:0 -t 3:8e00 -c 3:"LVM_PV_0" /dev/nvme0n1
```

### Disk 2
```bash
# 1. Create LVM Partition with remaining space (Type 8E00)
sgdisk -n 3:0:0 -t 3:8e00 -c 3:"LVM_PV_0" /dev/nvme1n1
```

## 2. Disk Formatting & LVM Setup
*Utilizing persistent labels to prevent asynchronous NVME enumeration swaps (`nvme0` vs `nvme1`).*

```bash
# System Disk (OS)
mkfs.fat -F 32 -n BOOT /dev/nvme0n1p1
mkfs.btrfs -L ROOT -f /dev/nvme0n1p2

# Data Disk (VMs & Heavy Files) - LVM Layout
pvcreate /dev/nvme1n1
vgcreate vg_data /dev/nvme1n1
lvcreate -L 400G -n lv_storage vg_data
mkfs.ext4 -L DATA /dev/mapper/vg_data-lv_storage
```

## 3. Btrfs Subvolume Initialization
```bash
mount /dev/disk/by-label/ROOT /mnt

# Create the top-level subvolumes
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@pkg
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@home_snapshots

umount /mnt
```

## 4. Mounting the File System
```bash
# Mount root with optimized performance flags
mount -o subvol=@,compress=zstd,noatime /dev/disk/by-label/ROOT /mnt

# Create target directories
mkdir -p /mnt/{boot,home,var/log,var/cache/pacman/pkg,.snapshots,home/.snapshots}

# Mount remaining subvolumes and boot partition
mount -o subvol=@home,compress=zstd,noatime /dev/disk/by-label/ROOT /mnt/home
mount -o subvol=@log,compress=zstd,noatime /dev/disk/by-label/ROOT /mnt/var/log
mount -o subvol=@pkg,compress=zstd,noatime /dev/disk/by-label/ROOT /mnt/var/cache/pacman/pkg
mount -o subvol=@snapshots,compress=zstd,noatime /dev/disk/by-label/ROOT /mnt/.snapshots
mount -o subvol=@home_snapshots,compress=zstd,noatime /dev/disk/by-label/ROOT /mnt/home/.snapshots
mount /dev/disk/by-label/BOOT /mnt/boot
```

## 5. Base System Installation
*Using the Linux-Zen kernel for performance and LTS as a fallback.*

```bash
pacstrap -K /mnt base base-devel linux-zen linux-zen-headers linux-lts linux-lts-headers linux-firmware nvim nano btrfs-progs lvm2 networkmanager snapper grub grub-btrfs inotify-tools xdg-user-dirs
```

## 6. System Configuration
```bash
# Generate fstab
genfstab -U /mnt >> /mnt/etc/fstab

# Enter chroot
arch-chroot /mnt
```

### 6.1 Hostname, Locale, and Keyboard
```bash
# Set Hostname
echo "ingGGRM-ArchLinux" > /etc/hostname

# Set Keyboard (US International for special characters/accents)
echo "KEYMAP=us-acentos" > /etc/vconsole.conf

# Set Locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### 6.2 User Creation
```bash
useradd -m -G wheel -s /bin/bash ingGGRM
passwd ingGGRM

# Enable sudo privileges (Uncomment %wheel ALL=(ALL:ALL) ALL)
EDITOR=nano visudo
```

### 6.3 Initramfs, Bootloader & Boot the System
```bash
# Edit mkinitcpio.conf to include LVM and BTRFS overlay hooks
nano /etc/mkinitcpio.conf
# Ensure HOOKS line looks like this:
# HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block lvm2 filesystems fsck grub-btrfs-overlayfs)

# Regenerate kernel images
mkinitcpio -P

# Install GRUB
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg

# Reboot to start the fresh system
reboot
```

## 7. Manual Snapshot & Rollback Infrastructure
*Ensuring strict manual control over system states without automated interference.*

```bash
# Need to temporary undo the snapshots subvolumes mount and delete the folders
sudo umount /.snapshots /home/.snapshots
sudo rmdir /.snapshots /home/.snapshots

# Crete the snapper configs (this will create the folders again)
sudo snapper -c root create-config /
sudo snapper -c home create-config /home

# We'll delete the new folders and recreate them for the mount of the snapshot subvolumes
sudo rmdir /.snapshots /home/.snapshots
sudo mkdir -p /.snapshots /home/.snapshots
# Now we'll use the fstab for remount the umounted subvolumes
sudo mount -a
```

```bash
# Populate xdg dirs in user home
xdg-user-dirs-update
# Create local bin directory for user scripts
mkdir -p /home/ingGGRM/.local/bin
```

### 7.1 `snap-create` (Manual Snapshot Creation)
```bash
cat << 'EOF' > /home/ingGGRM/.local/bin/snap-create
#!/bin/bash
# Matched Snapshot Creator with Sudo Check

# Ask for sudo upfront
sudo -v || exit 1

if [ -z "$1" ]; then
    echo "Usage: mksnap 'Description'"
    exit 1
fi

DESC="$1"
PAIR_ID="pair-$(date +%s)"

read -p "Snapshot /home as well? (y/N): " SNAP_HOME

echo "Creating root snapshot..."
sudo snapper -c root create --description "$DESC" --userdata "pair_id=$PAIR_ID"

if [[ "$SNAP_HOME" =~ ^[Yy]$ ]]; then
    echo "Creating matched home snapshot..."
    sudo snapper -c home create --description "$DESC [Linked to Root]" --userdata "pair_id=$PAIR_ID"
    echo "Success: Matched snapshots created with ID: $PAIR_ID"
else
    echo "Success: Root-only snapshot created."
fi

sudo grub-mkconfig -o /boot/grub/grub.cfg
EOF
```

### 7.2 `snap-rollback` (OverlayFS Writeable Recovery)
```bash
cat << 'EOF' > /home/ingGGRM/.local/bin/snap-rollback
#!/bin/bash
# Intelligence-driven Rollback with Sudo Check

# Ask for sudo upfront
sudo -v || exit 1

# 1. Mount Top-Level
mkdir -p /tmp/btrfs-root
sudo mount /dev/disk/by-label/ROOT /tmp/btrfs-root -o subvolid=5

# 2. Identify Current State
CURRENT_SUBVOL=$(grep -oP 'subvol=\K[^ ]+' /proc/cmdline)
echo "--- Rollback Diagnostic ---"
echo "Current Boot Subvolume: $CURRENT_SUBVOL"

if [[ $CURRENT_SUBVOL != *"snapshots"* ]]; then
    echo "Error: Not booted into a snapshot."
    sudo umount /tmp/btrfs-root && exit 1
fi

SNAP_NUM=$(echo "$CURRENT_SUBVOL" | grep -oP '(?<=snapshots/)\d+')
echo "Detected Root Snapshot Number: $SNAP_NUM"

# 3. Metadata Search
PAIR_ID=$(snapper -c root list --columns number,userdata | awk -v sn="$SNAP_NUM" '$1 == sn {print $0}' | grep -oP 'pair_id=\Kpair-[0-9]+')

# 4. Perform Root Rollback
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
echo "Rolling back Root (@) to snapshot #$SNAP_NUM..."
sudo mv /tmp/btrfs-root/@ /tmp/btrfs-root/@broken_root_$TIMESTAMP
sudo btrfs subvolume snapshot /tmp/btrfs-root/"$CURRENT_SUBVOL" /tmp/btrfs-root/@

# 5. Conditional Home Rollback
if [ -n "$PAIR_ID" ]; then
    echo "Found Linked Pair ID: $PAIR_ID"
    HOME_SNAP_NUM=$(snapper -c home list --columns number,userdata | grep "pair_id=$PAIR_ID" | awk '{print $1}')
    
    if [ -n "$HOME_SNAP_NUM" ]; then
        echo "Found matching Home snapshot #$HOME_SNAP_NUM. Rolling back @home..."
        sudo mv /tmp/btrfs-root/@home /tmp/btrfs-root/@broken_home_$TIMESTAMP
        sudo btrfs subvolume snapshot /tmp/btrfs-root/@home_snapshots/"$HOME_SNAP_NUM"/snapshot /tmp/btrfs-root/@home
    fi
fi

# 6. Cleanup
sudo umount /tmp/btrfs-root
echo "----------------------------------------------------"
echo "Rollback Complete. Reboot to finalize."
EOF
```

### 7.3 `snap-broken-remove` (Deep Janitor Cleanup)
```bash
cat << 'EOF' > /home/ingGGRM/.local/bin/snap-broken-remove
#!/bin/bash
# Deep-cleaning script with Sudo Check

# Ask for sudo upfront
sudo -v || exit 1

TEMP_MNT="/tmp/btrfs-clean"
mkdir -p "$TEMP_MNT"
sudo mount -t btrfs -o subvolid=5 /dev/disk/by-label/ROOT "$TEMP_MNT"

BROKEN_LIST=$(ls "$TEMP_MNT" | grep "^@broken_" 2>/dev/null)

if [[ -z "${BROKEN_LIST// }" ]]; then
    echo "No broken subvolumes found."
    sudo umount "$TEMP_MNT"
    exit 0
fi

echo "Detected broken systems: $BROKEN_LIST"
read -p "Permanently delete these items? (y/N): " CONFIRM

if [[ "$CONFIRM" =~ ^[Yy]$ ]]; then
    for item in $BROKEN_LIST; do
        echo "Cleaning $item..."
        CHILDREN=$(sudo btrfs subvolume list -o "$TEMP_MNT/$item" | awk '{print $NF}' | sort -r)
        for child in $CHILDREN; do
            sudo btrfs subvolume delete "$TEMP_MNT/$child"
        done
        sudo btrfs subvolume delete "$TEMP_MNT/$item"
    done
    echo "Cleanup complete."
fi

sudo umount "$TEMP_MNT"
EOF
```

```bash
# Make scripts executable and assign ownership
chmod +x /home/ingGGRM/.local/bin/*
chown -R ingGGRM:ingGGRM /home/ingGGRM/.local/bin
```

## 8. Power TTY Environment (Zsh)
```bash
# Install Zsh and modern CLI tools
pacman -S zsh zsh-completions zsh-autosuggestions zsh-syntax-highlighting exa bat fzf

# Apply the environment config
cat << 'EOF' >> /home/ingGGRM/.zshrc
export PATH="$HOME/.local/bin:$PATH"
HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=10000
setopt APPEND_HISTORY
setopt SHARE_HISTORY

alias ls='exa --icons --group-directories-first'
alias ll='exa -lh --icons --group-directories-first'
alias cat='bat --theme="base16" --style=plain'

source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
source /usr/share/fzf/key-bindings.zsh
source /usr/share/fzf/completion.zsh

autoload -Uz compinit && compinit
bindkey '^[[A' up-line-or-search
bindkey '^[[B' down-line-or-search
PROMPT='%F{cyan}%n%f@%F{blue}%m%f %F{yellow}%1~%f %# '
EOF
```
## 9. Giving User Data LVM Disk Management (/data)
*The rest of 1st disk and the whole 2nd disk were mapped into a single LVM partition with EXT4 fs for general storage, VMs, and large projects.*

```bash
# Give permissions to the user to manage /data directory files
sudo chown -R ingGGRM:ingGGRM /data
```

## 10. Hybrid Graphics Base (Intel + NVIDIA)
*Configuring the Acer Nitro 5 hardware stack (i5-10300H iGPU + GTX 1650 dGPU) for dynamic offloading using the `linux-zen` kernel.*

### 10.1 Driver Installation
*Using `nvidia-dkms` for compatibility across multiple kernels (Zen and LTS).*

```bash
# Install the graphics stack and PRIME offloading utilities
sudo pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils nvidia-settings \
lib32-opencl-nvidia libglvnd lib32-libglvnd \
vulkan-icd-loader lib32-vulkan-icd-loader \
mesa lib32-mesa mesa-utils vulkan-intel lib32-vulkan-intel \
intel-media-driver nvidia-prime
```

### 10.2 Kernel Modesetting (KMS) Integration
*Mandatory for Wayland compositors to properly control the display buffer without black screens.*

```bash
# 1. Update the GRUB bootloader parameters
sudo nano /etc/default/grub
# Append nvidia-drm.modeset=1 to GRUB_CMDLINE_LINUX_DEFAULT:
# GRUB_CMDLINE_LINUX_DEFAULT="... quiet splash nvidia-drm.modeset=1"

# Apply GRUB changes
sudo grub-mkconfig -o /boot/grub/grub.cfg

# 2. Force early loading of NVIDIA modules in initramfs
sudo nano /etc/mkinitcpio.conf
# Modify the MODULES array:
# MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)

# Regenerate kernel images
sudo mkinitcpio -P
```

### 10.3 Verification and Usage
```bash
# After rebooting, verify that the kernel modules are loaded:
lsmod | grep nvidia

# To run a specific application utilizing the NVIDIA GTX 1650, use the prime-run prefix:
prime-run nvidia-smi
```

---
*Note: At this stage, the base system architecture is complete, stable, and protected by the snapshot trinity. The system is now prepared for a Window Manager or Desktop Environment installation.*
