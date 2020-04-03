# Machine bootstrap

Unattended ssh based operating system installer

for Ubuntu 18.04-20.04, (WIP: Nixos, Debian 10),
with buildin recovery image, root storage on luks encrypted zfs,
initrd with ssh luks support, other specialized storage layouts.

**Usage:** to be executed on an linux liveimage/recoveryimage system connected via ssh.

It serves three use cases:
+ as an experimental Desktop/Laptop for getting experience with the setup
+ as a typical Rootserver (2xHD, headless)
+ as a home-nas/home-io headless server with one ssd and two attached spindle disks

Current Status: **Experimental** 

Some setups may work, most break under certain conditions.

## Features

+ **efi and legacy bios** boot compatible hybrid grub setup with grubenv support
+ **one or two disks** (will be automatically setup as mdadm &/ zfs mirror if two disks)
+ **root on luks encrypted zfs** / zfs mirror pool (encrypted storage at rest)
    + and **other common and less common**, easy to configure **storage setups**
+ ubuntu: modern initial ramdisk based on **dracut with ssh for remote unlock luks** on startup
+ **recovery system installation** (based on ubuntu 20.04 casper) on EFI partition
    + unattended cloud-init boot via custom squashfs with ssh ready to login
    + buildin scripts to mount/unmount root and update recovery boot parameter
+ **logging** of recovery and target system installation on the calling machine in directory ./log

### optional Features
+ luks encrypted **hibernate compatible swap** for eg. a desktop installation
+ ubuntu: **overlay fs and namespace uid/guid support on zfs** by building patched zfs-linux (frankenstein=true)
+ ubuntu: **saltstack provision run** at devop stage with states from salt-shared (eg. desktop)
+ **build a preconfigured livesystem image** usable for headless physical installation
    + resulting image is compatible as CD or USB-Stick with BIOS and EFI support
    + optional netplan for static or other non dhcp based ip configurations
    + execute `./machine-bootstrap/bootstrap.sh create-liveimage` to build image
    + copy `run/liveimage/bootstrap-0-liveimage.iso` to usbstick
    + use ssh with preconfigured key or physical terminal/console of livesystem for interaction
+ **encrypt sensitive data** in setup repository with git-crypt
    + git & git-crypt repository setup to store machine configuration inside a git repository

#### Example Configurations

+ a root server with custom network (eg. static ip)
    + add custom `netplan.yml`
+ a laptop with encrypted hibernation: 
    + add `storage_opts="--swap=yes"` to `machine-config.env`
+ a vm with a http proxy on its host:
    + add `http_proxy="http://proxyip:port"`to `machine-config.env`
+ install ubuntu focal instead of bionic:
    + add `distrib_codename=focal` to `machine-config.env`
+ install nixos instead of ubuntu:
    + add `distrib_id=Nixos` and `distrib_codename=19.09` to `machine-config.env`

##### Storage Examples

+ virtual machine with root on ext4
    + storage_opts="--boot=false --root-fs=ext4 --root-crypt=false"
+ virtual machine with encrypted root on ext4
    + storage_opts="--boot-fs=ext4 --root-fs=ext4"
+ virtual machine with encrypted lvm and root lv (30gb) on ext4
    + storage_opts="--boot-fs=ext4 --root-fs=ext4 --root-lvm=vgroot --root-lvm-vol-size=30720"
+ desktop: encrypted root and swap, boot and root on zfs, patched zfs for overlay support
    + storage_opts="--swap=true --frankenstein=true"
+ server: one or two encrypted disks, boot and root on zfs, patched zfs for overlay support
    + storage_opts="--frankenstein=true"
+ server: one or two encrypted disks with lvm storage (100gb) with root (25gb) and zfs on data (rest)
    + storage_opts="--root-fs=ext4 --root-size=102400 --root-lvm=vgroot --root-lvm-vol-size=25600" --data-fs=zfs"
+ server: one or two encrypted disks, root on zfs (100g), with data lvm storage vgdata
    + storage_opts="--frankenstein=true --root-size=102400 --data-lvm=vgdata --data-lvm-vol-size=25600" --data-fs=ext4"

## Preparation

Requirements:

+ Minimum RAM: **currently 4GB**, may become less again, was
    + frankenstein=no : 2GB RAM
    + frankenstein=yes: 4GB Ram (for compiling zfs in ram in recovery)
+ Minimum Storage
    + 10GB (no swap, console) 
    + \+ ~5GB (with swap for 4gb memory)
    + \+ ~5GB (full desktop installation)

### make a new project repository (eg. box)
```
mkdir box
cd box
git init
mkdir -p config log run
printf "# machine-bootstrap ignores\n/log\n/run\n" > .gitignore
git submodule add https://github.com/wuxxin/machine-bootstrap.git
git add .
git commit -v -m "initial config"
```

### optional: add an upstream
```
git remote add origin ssh://git@some.where.net/path/box.git
git push -u origin master
```

### optional: add files for a saltstack run (only ubuntu and debian)
```
mkdir -p salt/custom
cd salt
git submodule add https://github.com/wuxxin/salt-shared.git
cd ..
cat > config/top.sls << EOF
base:
  '*':
    - custom
EOF
for i in custom bootstrap; do
    cp machine-bootstrap/devop/${i}-pillar.sls config/${i}.sls
done
cp machine-bootstrap/devop/top-state.sls salt/custom/top.sls
touch salt/custom/custom.sls
git add .
git commit -v -m "add saltstack skeleton"
```

### optional: git-crypt config

```
git-crypt init
cat > .gitattributes <<EOF
**/secret/** filter=git-crypt diff=git-crypt
**/secrets/** filter=git-crypt diff=git-crypt
*secrets* filter=git-crypt diff=git-crypt
*secret* filter=git-crypt diff=git-crypt
*key filter=git-crypt diff=git-crypt
*keys filter=git-crypt diff=git-crypt
*id_rsa* filter=git-crypt diff=git-crypt
*id_ecdsa* filter=git-crypt diff=git-crypt
*id_ed25519* filter=git-crypt diff=git-crypt
*.sec* filter=git-crypt diff=git-crypt
*.key* filter=git-crypt diff=git-crypt
*.pem* filter=git-crypt diff=git-crypt
*.p12 filter=git-crypt diff=git-crypt
credentials* filter=git-crypt diff=git-crypt
csrftokens* filter=git-crypt diff=git-crypt
random_seed filter=git-crypt diff=git-crypt
EOF
git-crypt add-gpg-user user.email@address.org
git add .
git commit -v -m "add git-crypt config"
```

### configure machine

```
cat > config/machine-config.env <<EOF
# mandatory
sshlogin=root@1.2.3.4
hostname=box.local
firstuser=$(id -u -n)
# storage_ids=""

# optional
# http_proxy="http://192.168.122.1:8123" # default ""
# distrib_id="Nixos" # default "Ubuntu"
# distrib_codename="19.09-small" # default "bionic"
# recovery_autologin="true" # default "false"
# frankenstein="true" # default "false"
# devop_target="/home/$(id -u -n)"
# devop_user="$(id -u -n)"
# storage_opts=""
# [--reuse]
# [--log=        true|*false|<logsizemb,   default if true: 1024 mb>]
# [--cache=      true|*false|<cachesizemb, default if true: 59392 mb>]
# [--swap=       true|*false|<swapsizemb,  default if true: 1.25xRAM mb>]
# [--efi-size=   <efisizemb, default: 1600 mb>]
# [--boot=       true|*false|<bootsizemb,  default if true: 400 mb>]
# [--boot-fs=    *zfs|ext4|xfs]
# [--root-fs=    *zfs|ext4|xfs]
# [--root-lvm=   *""|<vgname>]
# [--root-lvm-vol-size= <volsizemb, default if lvm is true: 20480 mb>]
# [--root-crypt= *true|false]
# [--root-size=  *all|<rootsizemb>]
# [--data-fs=    *""|zfs|ext4|xfs|other]
# [--data-crypt= *true|false]
# [--data-lvm=   *""|<vgname>]
# [--data-lvm-vol-size= <volsizemb, default if lvm is true: 20480 mb>]
EOF

# copy current user ssh public key as authorized_keys
cat ~/.ssh/id_rsa.pub ~/.ssh/id_ed25519.pub \
    > config/authorized_keys

# list serial(s) of harddisk(s)
./machine-bootstrap/connect.sh temporary "ls /dev/disk/by-id/"

# add serial(s) to config, eg. filter all virtio (but no partitions)
echo $(printf 'storage_ids="'; for i in \
    $(./machine-bootstrap/connect.sh temporary "ls /dev/disk/by-id/" | \
        grep -E "^virtio-[^-]+$"); do \
    printf "$i "; done; printf '"') >> config/machine-config.env

# create disk.passphrase.gpg
# example: create a random diskphrase and encrypted with user gpg key
(x=$(openssl rand -base64 12); echo -n "$x" | \
    gpg --encrypt -r username@email.address) \
    > config/disk.passphrase.gpg

```

### optional: create a custom netplan.yml file

```
cat > config/netplan.yml << EOF
network:
    version: 2
    ethernets:
        all-en:
            match:
                name: "en*"
            dhcp4: true
        all-eth:
            match:
                name: "eth*"
            dhcp4: true
EOF
```

### optional: create a minimal Nixos configuration.nix

```
cat > config/configuration.nix << EOF
# Help is available in the configuration.nix(5) man page
{ config, pkgs, ... }:
{
  imports =
    [ # Include the results of the hardware scan and machine-bootstrap scripts
      ./hardware-configuration.nix
      ./machine-bootstrap-configuration.nix
    ];
}
EOF
```

### optional: make physical bootstrap-0 liveimage

create-liveimage creates an iso hybrid image usable as CD or usb-stick,
bootable via efi or bios.

this can be useful eg. if the target is headless,
or inside an emulator supplied with a live image preconfigured with ssh and other keys.

```
./machine-bootstrap/bootstrap.sh create-liveimage
# copy run/liveimage/bootstrap-0-liveimage.iso to usbstick
# boot target machine from usbstick
```

## Installation

installation is done in 4 steps:

+ 1 initial live system: partition disk, recovery install to EFI partition, reboot into recovery
+ 2 recovery live system: build patches, create boot & root, debootstrap, configure system, chroot into target
+ 3 recovery chroot target: configure system, kernel, initrd, install standard software, reboot into target
+ 4 target system: install and run saltstack

```
# test if everything needed is there
./machine-bootstrap/bootstrap.sh test
# if alls looks fine, run
./machine-bootstrap/bootstrap.sh execute all box.local
# logs of each step will be written to log/bootstrap-*.log 
# but log directory is in .gitignore and content won't be comitted
git add .
git commit -v -m "bootstrap run"
```

### optional: push committed changes upstream

```
git push -u origin master

```

## Maintenance

### connect to machine

+ connect to target machine running in the temporary liveimage, in the recovery system, in the initrd of the productiton system, or the production system
```
./machine-bootstrap/connect.sh temporary|recovery|initrd|system
```

+ connect to initrd, open luks disks, exit, machine will continue to boot
```
./machine-bootstrap/connect.sh initrdluks
```

+ connect to recovery, open luks disks, mount storage, prepare chroot, shell
```
./machine-bootstrap/connect.sh recoverymount
```

### switch next boot to boot into recovery (from running target system)
```
grub-reboot recovery
reboot
```

## Notes

### Limits, Contraints, Automatics

+ one or two disks only, if machine has more disks they have to be setup later

+ SWAP
    if using a SWAP partition, the swap partition will always be encrypted.
    Also ROOT should be encrypted in this case.
    This is because suspend to disk has all memory secrets written to disk,
    so any suspend would store all secrets in plaintext to disk.
    Any other usages of swap beside suspend to disk where encryption may not
    be desired, can be created as swap file using create_file_swap()

+ ZFS or LVM but not both on one partition
    on partittions ROOT and DATA, currently only either zfs or lvm can be used.
    if both are specified at the same time, the script will fail.

+ if ZFS is specified and two disks are available, partition will be used as zfs mirror instead of mdadm raid mirror, to let zfs manage the mirroring.

### GPT Layout

+ information leakage
    be aware that, the way gpt labels are setup in this script, they leak metadata of encrypted data, eg. raid_luks_lvm.vg0_ext4_ROOT leaks the information that behind the luks encrypted data is a lvm partition with a volume group named vg0 and the root file system is ext4 (but not how to read this data). if this is an issue, simply rename volume labels after bootstrap completed, mounts will still work because they are pointing to the uuid.

+ Partition Layout

Nr |Name(max 36 x UTF16)|Description|
---|---|---
6  | `BIOS,1,2`  | GRUB-BIOS boot binary partition, used to host grub code on gpt for bios boot
5  | `EFI,1,2`   | EFI vfat partition
   |             |   + dual efi & bios grub installation
   |             |   + recovery- kernel,initrd,fs
   |             |   + system- kernel,initrd
4  | `LOG,1,2`   | **optional** ZFS Log or other usages
3  | `CACHE,1,2` | **optional** ZFS Cache or other usages
2  | `[raid_]luks_SWAP,1,2`  | **optional** encrypted hibernation compatible swap
1  | `[raid_](zfs:ext4:xfs)_BOOT,1,2`  | **optional**,**deprecated** boot partition, unencrypted, kernel,initrd
0  | `[raid_][luks_][lvm.vg0_](zfs:ext4:xfs)_ROOT,1,2` | root partition
7  | `[raid_][luks_][lvm.vgdata_](zfs:ext4:xfs:other)_DATA,1,2` | **optional** data partition
