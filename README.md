# ZFSBootMenu Fedora

## To ensure easist operation ##
- Run one release behind the current
- Update security patches daily/weekly
- Find out whatever the source of truth is for the zfs linux version and compatibility with kernel version. Possibly automate that?
- Only update after a ZFS snapshot

[Linux home directory on ZFS](https://discourse.practicalzfs.com/t/linux-home-directory-on-zfs/1429/5)
From Jim Salter
What you do is, you don’t have an explicit mountpoint for the home dataset in each of your operating systems at all. 
You simply nest the home dataset for each operating system as a child of the root dataset for that OS.
**I use a separate dataset, but mounted beneath my ZBM root for the distro. So, eg:**
pool
  |---ROOT
       |---Ubuntu
       |     |---home
       |
       |---Fedora
             |---home

**This way, I have cleanly separated home directories so that one distro doesn’t conflict with another, even when both have my home directory mounted at /home/jrs–but I can also still choose to replicate, roll forward, clone, etc my home directory independently from the base distribution (and vice versa).**

If you boot Fedora, then pool/ROOT/Fedora gets mounted (by ZFSBootMenu) on /. 
This, in turn, means that pool/ROOT/Fedora/home gets mounted on /home, not because it has an explicitly set mountpoint (it doesn’t!) 
but because it’s automatically mounted directly beneath /pool/ROOT/Fedora as its child dataset.
Similarly, if you boot Ubuntu, ZBM mounts pool/ROOT/Ubuntu as /, and therefore /pool/ROOT/Ubuntu/home gets mounted beneath it as /home.
When you boot Fedora, Ubuntu’s datasets aren’t mounted at all, anywhere (unless you explicitly mount them at the command line temporarily) and vice versa.
This is in contrast to the ZBM-documentation-suggested method of keeping a separate home dataset under pool/home with an explictly ZFS-set mountpoint of /home. 
Doing it that way means you have the same “home” dataset regardless of which OS you boot, which can lead to problems, which is why I’m recommending not doing it their way in the first place.

If ZBM mounts pool/ROOT/fedora on /, then pool/ROOT/fedora/home is automounted as /home, and pool/ROOT/ubuntu/home is not mounted anywhere at all, just as pool/ROOT/ubuntu is not mounted anywhere at all.
If you import the pool into a different system and zfs get mountpoint on both pool/ROOT/ubuntu/home and pool/ROOT/fedora/home, you’ll see both are set to “inherit” rather than to any explicit mountpoint.
By contrast, if you set up a system the way ZBM recommends, with pool/home as a dataset with explicit mountpoint of /home, then should you import that pool into a different system (rather than booting into it with ZBM) you’ll see that zfs get mountpoint pool/home returns /home.
If you wanted TEMPORARY access to Fedora’s home directories while Ubuntu is booted and mounted, you’d do something along the lines of zfs mount pool/ROOT/fedora/home /tmp/home. 
That would mount all of Fedora’s home directories TEMPORARILY at /tmp/home, WITHOUT changing the ZFS property “mountpoint” permanently.


**Switch to root user**
```
sudo -i
```
```
source /etc/os-release
export ID
```
**Install updated ZFS packages**
```
rpm -e --nodeps zfs-fuse
```
```
dnf config-manager --disable updates
```
```
dnf install -y https://zfsonlinux.org/fedora/zfs-release-2-5$(rpm --eval "%{dist}").noarch.rpm
```
```
dnf install -y https://dl.fedoraproject.org/pub/fedora/linux/releases/${VERSION_ID}/Everything/x86_64/os/Packages/k/kernel-devel-$(uname -r).rpm
```
```
dnf install -y zfs gdisk
```
```
modprobe zfs
```
**Generate /etc/hostid**
```
zgenhostid -f 0x00bab10c
```
**Define disk variables**
- Single NVME
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
**Disk Prep**
- Wipe Partitions
```
zpool labelclear -f "$POOL_DISK"

wipefs -a "$POOL_DISK"
wipefs -a "$BOOT_DISK"

sgdisk --zap-all "$POOL_DISK"
sgdisk --zap-all "$BOOT_DISK"
```
**Create EFI Boot Partition**
```
sgdisk -n "${BOOT_PART}:1m:+512m" -t "${BOOT_PART}:ef00" "$BOOT_DISK"
```
**Create zpool Partition**
```
sgdisk -n "${POOL_PART}:0:-10m" -t "${POOL_PART}:bf00" "$POOL_DISK"
```
**ZFS Pool Creation**
- Create the zpool
- Encrypted
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
**Create Initial File Systems**
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
**Export, then re-import with a temp mountpoint of /mnt**
- Encrypted
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
**Verify**
```
mount | grep mnt
```
**Update device symlinks**
```
udevadm trigger
```
**Install Fedora**
```
mkdir /run/install
mount /dev/mapper/live-base /run/install
```
```
rsync -pogAXtlHrDx \
 --stats \
 --exclude=/boot/efi/* \
 --exclude=/etc/machine-id \
 --info=progress2 \
 /run/install/ /mnt
```
**Copy Files into the new Fedora Install**
- Encrypted
```
mv /mnt/etc/resolv.conf /mnt/etc/resolv.conf.orig
```
```
cp /etc/hostid /mnt/etc
```
```
cp -L /etc/resolv.conf /mnt/etc
```
```
mkdir -p /mnt/etc/zfs
```
```
cp /etc/zfs/zroot.key /mnt/etc/zfs
```
**Chroot into new OS**
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
**ZFS Configuration**
- Configure Dracut to load ZFS support
- Encrypted
```
cat << EOF > /etc/dracut.conf.d/zol.conf
nofsck="yes"
add_dracutmodules+=" zfs "
omit_dracutmodules+=" btrfs "
install_items+=" /etc/zfs/zroot.key "
EOF
```
**Install required Packages**
```
source /etc/os-release
```
```
rpm -e --nodeps zfs-fuse
```
```
dnf config-manager --disable updates
```
```
dnf install -y https://dl.fedoraproject.org/pub/fedora/linux/releases/${VERSION_ID}/Everything/x86_64/os/Packages/k/kernel-devel-$(uname -r).rpm
```
```
dnf --releasever=${VERSION_ID} install -y \  https://zfsonlinux.org/fedora/zfs-release-2-5$(rpm --eval "%{dist}").noarch.rpm
```
```
dnf install -y zfs zfs-dracut
```
```
dnf config-manager --enable updates
```
**Regenerate initramfs**
```
dracut --force --regenerate-all
```
**Install and configure ZFSBootMenu**
- Encrypted
```
zfs set org.zfsbootmenu:commandline="quiet rhgb" zroot/ROOT
```
```
zfs set org.zfsbootmenu:keysource="zroot/ROOT/${ID}" zroot
```
**Create a vfat filesystem**
```
mkfs.vfat -F32 "$BOOT_DEVICE"
```
**Create an fstab entry and mount**
```
cat << EOF >> /etc/fstab
$( blkid | grep "$BOOT_DEVICE" | cut -d ' ' -f 2 ) /boot/efi vfat defaults 0 0
EOF
```
```
mkdir -p /boot/efi
mount /boot/efi
```
**Install ZFSBootMenu**
- Prebuilt
```
mkdir -p /boot/efi/EFI/ZBM
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI
```
**Configure EFI Boot entries**
```
efibootmgr -c -d "$BOOT_DISK" -p "$BOOT_PART" \
  -L "ZFSBootMenu (Backup)" \
  -l '\EFI\ZBM\VMLINUZ-BACKUP.EFI'

efibootmgr -c -d "$BOOT_DISK" -p "$BOOT_PART" \
  -L "ZFSBootMenu" \
  -l '\EFI\ZBM\VMLINUZ.EFI'
```
**Reset resolv.conf**
```
mv /etc/resolv.conf.orig /etc/resolv.conf
```
**Prepare for first boot**
- Exit the chroot
- Unmount everything
```
exit
```
```
umount -n -R /mnt
```
**Export the zpool and reboot**
```
zpool export zroot
reboot
```

