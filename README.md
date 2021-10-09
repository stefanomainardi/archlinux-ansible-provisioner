# Archlinux ansible provisioner

This is a guide and a set of ansible roles to provision an Archlinux
installation, please be aware this is made just for personal use, use it at your own risk.

## Pre-requisites (manual steps)

#### Activate SSH

I like to install Archlinux using SSH, you can use this guide: https://wiki.archlinux.org/title/Install_Arch_Linux_via_SSH

Basically it is just `systemctl enable sshd && passwd`.

#### Install ansible and git

In order to use the provisioner we need to install `ansible` `git` and `make`, as we are going 
to use the provisioner to bootstrap the entire system.

```
mount -o remount,size=1G /run/archiso/cowspace
pacman -Sy --noconfirm ansible git make
```

The first command is needed because the root partition from live USB is just
256M which is not enough to install other dependencies.

### Disk layout and partitioning

Let me start by mentioning this guide: https://github.com/egara/arch-btrfs-installation
which is definetely my main source of truth for the following steps.

I want to use a BTRFS filesystem with the following layout:

```
nvme0n1 (Volume)
|
|
- @ (Subvolume - It will be the current /)
- @home (Subvolume - it will be the current /home)
- @snapshots (Subvolume -  It will contain the root snaphosts)
- @home.snapshots (Subvolume -  It will contain the home directories snaphosts)
```

I don't know if this layout is the best choice, but AFAIK this is the simplest
way to start and most compatible layout with tools like snapper or timeshift.

What i've used as a reference:

* https://forum.manjaro.org/t/default-btrfs-mount-options-and-subvolume-layout/43250/33
* https://www.reddit.com/r/archlinux/comments/fkcamq/noob_btrfs_subvolume_layout_help/fks5mph/?utm_source=share&utm_medium=web2x&context=3
* https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Layout
* https://www.jwillikers.com/btrfs-layout
* https://en.opensuse.org/SDB:BTRFS#Default_Subvolumes
* https://wiki.archlinux.org/title/User:M0p/LUKS_Root_on_Btrfs

This is the most flexible way to keep the things well separated and giving me the chance
to automatically snapshot the root or home volumes separately and using `grub-btrfs` 
to automatically from snapshots (TO BE TESTED).

This is my current disk layout (using `cfdisk`):

```
                                                      Disk: /dev/nvme0n1
                                   Size: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
                                 Label: gpt, identifier: 4BAEBBEA-64DB-4DE6-A7F2-04E8972BF153

    Device                                Start                 End             Sectors           Size Type
>>  /dev/nvme0n1p1                         2048              487423              485376           237M EFI System              
    /dev/nvme0n1p2                       487424          1139156991          1138669568           543G Linux filesystem
    /dev/nvme0n1p3                   1139156992          1206265855            67108864            32G Linux swap
    /dev/nvme0n1p4                   1206265856          1881548799           675282944           322G Linux filesystem
    /dev/nvme0n1p5                   1881548800          1953525134            71976335          34.3G Linux swap
```

I will use `/dev/nvme0n1p4` and `/dev/nvme0n1p5` respectively for `/` and `swap` partitions.
The rest is a Ubuntu 20.04 LTS installation. I'll share the EFI partition too between the two distros.

#### BTRFS installation

We are going to create btrfs volumes and subvolumes:

```
mkfs.btrfs -L arch /dev/nvme0n1p4
mount /dev/nvme0n1p4 /mnt

# This is a layout made to be compatible with timeshift and similare to the Manjaro one (https://www.reddit.com/r/archlinux/comments/fkcamq/comment/fks5mph/?utm_source=share&utm_medium=web2x&context=3)
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@home.snapshots
```

Next, create all the directories needed and mount all the partitions (/boot/efi included) in order to start the installation:

```
umount /mnt

# Mount root and create default directories.
mount -o noatime,compress=zstd,subvol=@ /dev/nvme0n1p4 /mnt
mkdir /mnt/{home,boot}
mkdir /mnt/boot/efi
mkdir -p /mnt/mnt/allvolumes

# Mount efi partition.
mount /dev/nvme0n1p1 /mnt/boot/efi

# Mount the subvolumes
mount -o noatime,compress=zstd,subvol=@home /dev/nvme0n1p4 /mnt/home
mount -o noatime,compress=zstd,subvol=/ /dev/nvme0n1p4 /mnt/mnt/allvolumes
```
> Please note that we are mounting btrfs with compression enabled to reduce writes (and ssd lifespan) 
and performance [here](https://wiki.archlinux.org/title/btrfs#Compression) and [here](https://fedoraproject.org/wiki/Changes/BtrfsByDefault#Compression) some refs.

Done, we can now run the provisioner.

## Provisioner

Start by cloning the repo:

```bash
git clone https://github.com/paolomainardi/archlinux-ansible-provisioner.git provisioner
cd provisioner
```

Now we run the first part of the installation, which will run `pacstrap`
and some other installations tasks:

```bash
make bootstrap install-packages
```

Now that the process if finished you can setup the password for the created user:

```
arch-chroot /mnt passwd <your-user>
```
