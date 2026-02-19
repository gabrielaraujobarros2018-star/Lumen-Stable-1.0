# Creating a Full Source Code .img File for Lumen OS

================================================

## Overview

--------

Create a raw .img file that holds your entire Linux distro source tree (e.g., Lumen OS) without any stripping, compression, or data loss. The image uses ext4 to preserve all permissions, symlinks, hardlinks, ACLs, and extended attributes. Optionally load the entire image into RAM for fast access.

Target use: Development/testing on Moto Nexus 6 or embedded systems where source access needs to be blazing fast in memory.

## Prerequisites

-------------

- Check source tree size: du -sh /path/to/lumen-os-source

- Ensure enough disk space (2x source size recommended)

- Root privileges (sudo)

### Step 1: Create Raw Image File

-----------------------------

`dd if=/dev/zero of=lumen-source-full.img bs=1M count=10240`

(Example for 10GB image - adjust count= based on your source size)

Creates a sparse 10GB file ready for formatting.

### Step 2: Attach Loop Device

--------------------------

```
sudo losetup -fP lumen-source-full.img  # e.g., attaches to /dev/loop0

sudo losetup -a  # Verify

-P enables partition scanning if needed later.
```

### Step 3: Format with ext4

------------------------

`sudo mkfs.ext4 -L "LumenSource" /dev/loop0`

ext4 preserves all Unix filesystem attributes (unlike FAT32).

### Step 4: Mount and Copy Source Tree

----------------------------------

```
sudo mkdir -p /mnt/lumen-img

sudo mount /dev/loop0 /mnt/lumen-img
```

# Critical: Use rsync with these flags for *exact* preservation

`sudo rsync -aHAX --info=progress2 /path/to/lumen-os-source/ /mnt/lumen-img/`

Flags explained:

- -a: Archive mode (permissions, times, links, etc.)

- -H: Preserve hard links

- -A: Preserve ACLs

- -X: Preserve extended attributes

- Trailing / on source copies *contents* without creating extra root dir

### Step 5: Unmount and Detach

--------------------------

```
sudo umount /mnt/lumen-img

sudo losetup -d /dev/loop0  # Replace with your loop device

sudo rmdir /mnt/lumen-img
```

Verification

------------

## Mount read-only to check

```
sudo mkdir /mnt/check

sudo mount -o ro /dev/loop0 /mnt/check  # Or mount the .img directly

ls -la /mnt/check  # Should match your source exactly

du -sh /mnt/check  # Should match original size

sudo umount /mnt/check
```

## RAM Loading (Boot-Time or Runtime)

----------------------------------

Runtime (on running Lumen OS):

# Create tmpfs ramdisk (match .img size)

`sudo mount -t tmpfs -o size=10G tmpfs /mnt/ramdisk`

# Copy image to RAM

`sudo dd if=lumen-source-full.img of=/mnt/ramdisk/source.img bs=1M status=progress`

# Mount from RAM

```
sudo losetup /dev/loop1 /mnt/ramdisk/source.img

sudo mkdir /lumen-source

sudo mount /dev/loop1 /lumen-source

cd /lumen-source  # Now compile/edit source at RAM speeds!
```

Boot-Time (initramfs hook for Lumen OS):

```
#!/bin/sh

## Parse kernel cmdline: 

ramdisk_size=10G img_path=/boot/lumen-source.img mount_point=/source

RAMDISK_SIZE=${ramdisk_size:-10G}

IMG_PATH=${img_path:-/boot/lumen-source-full.img}

MOUNT_POINT=${mount_point:-/lumen-source}

mount -t tmpfs -o size=$RAMDISK_SIZE tmpfs /mnt/ramdisk

dd if=$IMG_PATH of=/mnt/ramdisk/source.img bs=1M

losetup /dev/loop1 /mnt/ramdisk/source.img

mkdir -p $MOUNT_POINT

mount /dev/loop1 $MOUNT_POINT

echo "Lumen OS full source loaded to RAM at $MOUNT_POINT"
```

## Optimization Notes for Lumen OS

-------------------------------

- Moto Nexus 6 (ARMv7a): 3GB RAM total—allocate ~2GB max for source ramdisk to leave headroom.

- No stripping: rsync -aHAX + ext4 = bit-for-bit source tree replica.

- Compression alternative: If space critical, use mksquashfs instead of raw ext4, but loses some attributes.

- Git integration: Source includes .git/—works perfectly for git status/pull from RAM.

## Troubleshooting

---------------

Issue                                    | Solution

-----------------------------------------|---------------------------------------------------

"wrong fs type" on mount                | Verify mkfs.ext4; check file lumen-source.img shows "ext4"

Permissions wrong                       | Always use sudo rsync -aHAX; never cp

Loop device busy                        | sudo losetup -d /dev/loopX; check fuser -m /dev/loopX

RAM too small                           | Reduce count= in dd; use zram for compression

Sparse image not growing                | Normal—expands on first write; use fallocate for pre-allocation

Result: lumen-source-full.img contains your entire Lumen OS source tree exactly as-is, ready for flashing, distribution, or RAM loading.
