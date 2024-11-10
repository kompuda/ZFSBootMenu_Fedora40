# ZFSBootMenu_Fedora40

Cut Down Code to Cut Down Copy Pasta
```
sudo -i
```
```
dmesg | grep -i efivars
```
```
source /etc/os-release
```
```
export ID
```
```
apt update
```
```
apt install debootstrap gdisk zfsutils-linux
```
```
zgenhostid -f 0x00bab10c
```
Single NVME
```
export BOOT_DISK="/dev/nvme0n1"
export BOOT_PART="1"
export BOOT_DEVICE="${BOOT_DISK}p${BOOT_PART}"
```
```
export POOL_DISK="/dev/nvme0n1"
export POOL_PART="2"
export POOL_DEVICE="${POOL_DISK}p${POOL_PART}"
```
```
zpool labelclear -f "$POOL_DISK"

wipefs -a "$POOL_DISK"
wipefs -a "$BOOT_DISK"

sgdisk --zap-all "$POOL_DISK"
sgdisk --zap-all "$BOOT_DISK"
```
```
sgdisk -n "${BOOT_PART}:1m:+512m" -t "${BOOT_PART}:ef00" "$BOOT_DISK"
```
```
sgdisk -n "${POOL_PART}:0:-10m" -t "${POOL_PART}:bf00" "$POOL_DISK"
```
```
echo 'SomeKeyphrase' > /etc/zfs/zroot.key
```
```
chmod 000 /etc/zfs/zroot.key
```
```
zpool create -f -o ashift=12 \
 -O compression=lz4 \
 -O acltype=posixacl \
 -O xattr=sa \
 -O relatime=on \
 -O encryption=aes-256-gcm \
 -O keylocation=file:///etc/zfs/zroot.key \
 -O keyformat=passphrase \
 -o autotrim=on \
 -o compatibility=openzfs-2.1-linux \
 -m none zroot "$POOL_DEVICE"
```
```
zfs create -o mountpoint=none zroot/ROOT
```
```
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/${ID}
```
```
zfs create -o mountpoint=/home zroot/home
```
```
zpool set bootfs=zroot/ROOT/${ID} zroot
```
```
zpool export zroot
```
```
zpool import -N -R /mnt zroot
```
```
zfs load-key -L prompt zroot
```
```
zfs mount zroot/ROOT/${ID}
```
```
zfs mount zroot/home
```
Verify
```
mount | grep mnt
```
```
udevadm trigger
```
Install Ubuntu 24.04LTS
```
debootstrap noble /mnt
```
Copy files into the new install
```
cp /etc/hostid /mnt/etc/hostid
```
```
cp /etc/resolv.conf /mnt/etc/
```
```
mkdir /mnt/etc/zfs
```
```
cp /etc/zfs/zroot.key /mnt/etc/zfs
```
Chroot into new OS
```
mount -t proc proc /mnt/proc
```
```
mount -t sysfs sys /mnt/sys
```
```
mount -B /dev /mnt/dev
```
```
mount -t devpts pts /mnt/dev/pts
```
```
chroot /mnt /bin/bash
```
```
echo 'YOURHOSTNAME' > /etc/hostname
```
```
echo -e '127.0.1.1\tYOURHOSTNAME' >> /etc/hosts
```
```
passwd
```
```
cat <<EOF > /etc/apt/sources.list
# Uncomment the deb-src entries if you need source packages

deb http://us.archive.ubuntu.com/ubuntu/ noble main universe multiverse restricted 
# deb-src http://us.archive.ubuntu.com/ubuntu/ noble main universe multiverse restricted

deb http://us.archive.ubuntu.com/ubuntu/ noble-updates main universe multiverse restricted
# deb-src http://us.archive.ubuntu.com/ubuntu/ noble-updates main universe multiverse restricted

deb http://us.archive.ubuntu.com/ubuntu/ noble-security main universe multiverse restricted
# deb-src http://us.archive.ubuntu.com/ubuntu/ noble-security main universe multiverse restricted

deb http://us.archive.ubuntu.com/ubuntu/ noble-backports main universe multiverse restricted
# deb-src http://us.archive.ubuntu.com/ubuntu/ noble-backports main universe multiverse restricted
EOF
```
```
apt update
```
```
apt upgrade
```
Must install to get Networking and basic text editing capabilities from Nano
```
apt install isc-dhcp-client bind9-dnsutils nano
```
```
apt install --no-install-recommends linux-generic locales keyboard-configuration console-setup
```
```
dpkg-reconfigure locales tzdata keyboard-configuration console-setup
```
Install Desktop Environment
```
apt install ubuntu-desktop
```
ZFS Configuration
```
apt install dosfstools zfs-initramfs zfsutils-linux
```
```
systemctl enable zfs.target
```
```
systemctl enable zfs-import-cache
```
```
systemctl enable zfs-mount
```
```
systemctl enable zfs-import.target
```
Encrypted
```
echo "UMASK=0077" > /etc/initramfs-tools/conf.d/umask.conf
```
```
update-initramfs -c -k all
```
```
zfs set org.zfsbootmenu:commandline="quiet" zroot/ROOT
```
```
zfs set org.zfsbootmenu:keysource="zroot/ROOT/${ID}" zroot
```
```
mkfs.vfat -F32 "$BOOT_DEVICE"
```
```
cat << EOF >> /etc/fstab
$( blkid | grep "$BOOT_DEVICE" | cut -d ' ' -f 2 ) /boot/efi vfat defaults 0 0
EOF
```
```
mkdir -p /boot/efi
```
```
mount /boot/efi
```
```
apt install curl
```
```
mkdir -p /boot/efi/EFI/ZBM
```
```
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
```
```
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI
```
```
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```
```
apt install efibootmgr
```
```
efibootmgr -c -d "$BOOT_DISK" -p "$BOOT_PART" \
  -L "ZFSBootMenu (Backup)" \
  -l '\EFI\ZBM\VMLINUZ-BACKUP.EFI'
```
```
efibootmgr -c -d "$BOOT_DISK" -p "$BOOT_PART" \
  -L "ZFSBootMenu" \
  -l '\EFI\ZBM\VMLINUZ.EFI'
```
```
exit
```
```
umount -n -R /mnt
```
```
zpool export zroot
```
```
reboot
```


