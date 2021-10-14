---
title: "SCUSA: Contest Disk Images"
date: 2021-02-01T20:00:00-05:00
draft: false
---

For a number of years I have helped to run the ACM-ICPC South Central USA regional programming
competition as the tech chair. As such one of my responsibilities is building the OS that the
contestants use on the day of the competition.

<!--more-->

The competition is spread out between sites in Louisiana, Texas, and Oklahoma so all of the sites
need to set up their own computer labs using our contest image.

## LiveCD-based Image

We have always used a RHEL-derivative as the base but in the recent years we have moved to using a
LiveCD-based install on a USB drive. This makes it easy to boot an entire lab of computers into the
contest image for the weekend without modifying the OS on the system disk of the machine.

The LiveCD makes use of a SquashFS read-only layer and a read-write layer in RAM. SquashFS is
perfect for reducing the overall size of the image which gets quite large since we ship a number of
programming languages and IDEs in the desktop environment. In order to prevent data loss in the
event of a kernel panic or power event, we have a partition on the USB drive that is set aside for
`/home`.

Switching to CentOS 8 let us to mount the root SquashFS with a R/W layer in RAM using OverlayFS
instead of device mapper. Device mapper is not ideal because it uses a snapshot volume as the RW
layer which can become full if too many blocks are written to the snapshot. The snapshot volume is
also difficult to monitor but with OverlayFS we can monitor the R/W layer that is mounted from a
tmpfs path.

## Simplifying the Site Administrator Experience

In the past, the process for the site administrators to install the LiveCD to USB was overly
complex. When we first started doing the LiveCD a number of years ago there was not much
cross-platform tooling for burning USB disks. There were projects like `unetbootin` but they had a
very specific way of creating the bootable USB which interfered with how we wanted a separate /home
partition on the drive. We settled on distributing a LiveCD ISO that each site would download and
block copy to a USB drive (using whatever was available on any OS such as `dd`). This would allow
the site organizers to boot into our OS but since it was just a block copy we did not have a R/W
partition for `/home`. However once booted into this live environment the user could run a shell
script that would take a second USB drive and make it bootable with the correct partitions.

We did this so that we did not have to maintain a disk formatting script for all three major OSes
that the site administrators may want to use. By having them boot into our ISO we would only have to
maintain a script for Linux and we could control and test the userspace tools ourselves easily.

While this made it easier for **us** to maintain, it was a huge drag on the **site administrators**
for the competition. The multi-step process of making a single USB image and then booting it into a
new environment to make more USB images was complex and prone to error and we had to write a lot of
check logic to make sure that each site had the right version and followed the right procedure to
make the USB images.

While reviewing this entire process, we decided that we needed to cut out the multi-step process:

1. Burning the ISO image to a USB drive
1. Booting a contest machine with that USB drive
1. Using that OS to burn *a second USB drive*
1. Boot contest machine with *the second USB drive* for competition

Instead we needed it to look like:

1. Burn disk image to USB drive
1. Boot contest machine with USB drive for competition

In order to achieve our simplified set of operations we moved from providing a ISO image to just
providing a compressed disk image. One of the nice side effects of the Raspberry PI movement is that
there are now an abundance of high quality tools that will take a disk image and copy it to
removable media such as [Balena Etcher](https://www.balena.io/etcher/).

## Building an Image

We have used the [livemedia-creator](https://weldr.io/lorax/livemedia-creator.html) toolchain for
building the ISO image and were familiar with the intricacies of how it took a kickstart file and
built the complete ISO image so we wanted to stick with that as our base.

### Step 1: Build the Image Tarball

The `livemedia-creator` script orchestrates the Anaconda installer to bootstrap a system install from
a kickstart file. By specifying `--make-tar` it just outputs a tarball we can use for the next step.

```bash
livemedia-creator --no-virt --make-tar --tmp=/work/livemedia --resultdir=/work/output
```

This made it trivial (but time consuming, a install takes about 30 min if it has to go download all of
the RPMs) to get our base OS configured.

### Step 2: Make R/O SquashFS Volume

We can then make our SquashFS volume. For development we set ` -Xcompression-level 1`  because it
significantly speeds up the process but for the production image, omitting this argument does the
default of highest compression level for gzip.

```bash
mkdir /work/tmproot
tar -xzf /work/livemedia/root.tar.gz -C /work/tmproot/
mksquashfs /work/tmproot /work/output/squashfs.img
```

### Step 3: Put It All Together

The last step is to build the disk image, setting up the partitions, boot loader, and copying the
necessary files including the SquashFS volume to the boot partition.

```bash
truncate -s 8G /work/output/boot.img

parted --script /work/output/boot.img mklabel msdos mkpart primary fat32 2MB 4GB

parted --script /work/output/boot.img set 1 boot on
parted --script /work/output/boot.img mkpart primary 4GB 8GB
kpartx -vas /work/output/boot.img
mkfs.vfat -F 32 -n SCUSA-LIVE /dev/mapper/loop0p1
mkfs.ext4 -L SCUSA-HOME /dev/mapper/loop0p2
MOUNT_IMG=$(mktemp -d)
mount /dev/mapper/loop0p1 $MOUNT_IMG

grub2-install --boot-directory=$MOUNT_IMG /dev/loop0
cat <<'EOF' > $MOUNT_IMG/grub2/grub.cfg
set default="0"

function load_video {
  insmod efi_gop
  insmod efi_uga
  insmod video_bochs
  insmod video_cirrus
  insmod all_video
}

load_video
set gfxpayload=keep
insmod gzio
insmod part_gpt
insmod ext2

set timeout=5

submenu 'SCUSA Contest Image' {
    menuentry 'Start' --class fedora --class gnu-linux --class gnu --class os {
        linux /vmlinuz root=live:LABEL=SCUSA-LIVE rd.live.image rd.live.dir=/ rd.live.overlay.overlayfs=1
        initrd /initrd.img
    }
}
EOF

cp /work/output/squashfs.img $MOUNT_IMG
cp /work/tmproot/boot/vmlinuz-*.x86_64 $MOUNT_IMG/vmlinuz
cp /work/tmproot/boot/initramfs-*.x86_64 $MOUNT_IMG/initrd.img

umount $MOUNT_IMG
kpartx -vd /work/output/boot.img
```

## Testing the Disk Image

For testing, it is easy to convert the disk image into a VMDK for loading into VMWare Fusion or
VirtualBox:

```bash
qemu-img convert -O vmdk /work/output/boot.img /work/output/images/boot.vmdk
```

## Conclusion

Overall it has been a fun project to dig into the LiveCD process and build it all from scratch
(relatively speaking, we are still `yum install kernel` after all). All of the SquashFS/OverlayFS
features are built right into stock Dracut modules which makes the whole process easy to build and
maintain.
