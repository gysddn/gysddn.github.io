---
title: "Booting with Limine"
date: 2023-05-20T13:04:49+03:00
draft: false
---

Let's boot the operating system with a simpler bootloader!

A couple of days ago, I was informed of a modern and simpler bootloader named "Limine" and I just wanted
to give it a go. This is the implementation of the boot protocol with the same name. It aims to do
what conventional ones do but in a simpler way, or so I'm told.

Originally, the project was booting on BIOS using grub2. It had a simple multiboot(v1) header, and the
rest was done with grub-mkrescue.

I would like to change two things:

- Switch to UEFI
- And use Limine bootloader

### Switching to UEFI

My way to test the operating system is qemu. Qemu uses bios, there is no support for UEFI.
But it does allow you to specify the bios firmware, so all one has to do is to just use a UEFI
bios firmware when running qemu. As for where to find a such firmware, there is project named
OVMF (Open Virtual Machine Firmware) that develops firmwares to enable UEFI in virtual devices.
These firmware files are in the related packages and can be used with qemu.

Let's install that package and find the file:

```bash
sudo pacman -S edk2-ovmf
pacman -Ql edk2-ovmf | grep OVMF.md
```

This will output something like this:

```bash
edk2-ovmf /usr/share/edk2/ia32/OVMF.fd
edk2-ovmf /usr/share/edk2/x86/OVMF.fd
```

There are to files, one for x86 and x86-64. The approrpiate one can now be used with the "-bios"
flag:

```bash
qemu-system-x86_64 -bios /usr/share/edk2/x86/OVMF.fd  -hda disk.img
```

### Using Limine Bootloader

The kernel uses multiboot (v1) and is able to boot with that. Since Limine also supports this protocol
it must be possible to try and boot it without making any changes to it.

While using grub, the "grub-mkrescue" took care of creating the iso from a sysroot directory, now
with Limine this process is going to be less automated.

UEFI requires an ESP partition formatted with FAT32 to be present in the disk with a GPT table. This partition is where
the bootloader resides, the UEFI will find the appropriate bootloader for the current architecture inside
the "EFI/BOOT" folder. For example the file "BOOTX64.efi" will be loaded and executed by UEFI on a
x86-64 machine, which is the file we will put in there, and it is the prepared UEFI executable file of Limine.

Let's start by creating an empty image to work on:

```
dd if=/dev/zero of=disk.img bs=1M count=16
```

This command will copy 16 Megabyte of zeros into a file named disk.img. It's just zeros.

```bash
hexdump disk.img
```

Will output:

```
0000000 0000 0000 0000 0000 0000 0000 0000 0000
*
1000000
```

Unrelated to the task at hand, but since that above is the case, it's all zeroes. There is a better way
to create this image, so I'm told by a nice fella. Instead of copying zeroes, you can just move the
cursor to an offset when writing and all that's preceding it will just be holes and information will
be *represented* by a metadata rather than physical data. This will be a sparse file and it's way more
efficient. To do that is simple, don't copy anything, just seek the cursor at the offset you desire.

```
dd if=/dev/zero of=disk.img bs=1M count=0 seek=16
```

Same effect.

After that, this hypothetical disk needs to be formatted with a GPT table. It is possible to do this
in many ways, one of them is using "fdisk", one can just open the file like a device:

```
fdisk disk.img
```

The program will open up a prompt to work on the image.

```
Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x5bbd477f.
Command (m for help):
```

First off, it needs to have a GPT partition table. Then it needs the boot partition itself to
be created and set to "EFI System". The command flow is 'g', 'n' (entering to all defaults will suffice)
and 't' with 1. Once these steps are completed, there is going to be one partition that will hold everything
as well as the bootloader itself.

That's the partition table, command 'p' will print it:

```
Disk disk.img: 16 MiB, 16777216 bytes, 32768 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: FE25C548-D7FF-D043-B3C3-48833C2F40EA

Device     Start   End Sectors Size Type
disk.img1   2048 32734   30687  15M EFI System
```

The disk now has its partition table configured. It is possible to move on after this point, our firmware
will be able to boot if the rest of the stages are complete as well, but just in case, if at one point
this image gets put into a real device to boot it's going to need the stage 2 deployed in it. This has to do
with the limitations of how BIOS operates but all there is to do is to let "limine-deploy" embed stage 2
into the image.

```
git clone --depth 1 -b v4.x-branch-binary https://github.com/limine-bootloader/limine.git
make -C limine
./limine/limine-deploy disk.img
```

Next step is to make a filesystem on the ESP partition, particularly FAT32. To do this, the "mkfs.fat" tool
will be used but this tool requires a device to operate on. Let's setup a loopback device:

```
sudo losetup -Pf --show disk.img
```

This will setup a loopback device with its partitions, it will find the first available loopback device
to use and will print the name of it, something like "/dev/loop1".

The disk and so the device has exactly one parition, so when the device is up there will be two files:

```
/dev/loop1
/dev/loop1p1
```

Let's make the filesystem on /dev/loop1p1:

```
sudo mkfs.fat -F 32 /dev/loop1p1
```

With that done, the filesystem will be ready. That means one can freely mount it and use it like a normal
FAT32 filesystem. And in fact, this needs to be done. It is time to put the files in there.

First to mount the device partition:

```
sudo mount /dev/loop1p1 /mnt
```

The first file needed is of course the bootloader executable BOOTX64.efi, it needs to go under EFI/BOOT.
This file is in the cloned git repository. Let's create those directories and put it in there:

```
mkdir -p /mnt/EFI/BOOT
cp limine/BOOTX64.efi /mnt/EFI/BOOT
```

The other files needed files are the kernel itself and limine.cfg file. Optionally, limine.sys file
for the same reason "limine-deploy" was used.

Since everything is simple, so will be the configuration file.

```
TIMEOUT=3

:OS
PROTOCOL=multiboot1
KERNEL_PATH=boot:///kernel
```

Here, timeout is the time before Limine boots the default selection, ":OS" is the name of the entry,
protocol is the booting protocol, and kernel path is where the kernel executable is, here "boot:///" is
the partition where boot files are.

```
sudo cp -v ${KERNEL_PATH} ${LIMINE_CFG} limine.sys mnt/
```

This marks the end of this process. All there is left to do is to 'sync' the changes to the devices, unmount and
detach the loopback device. The file "disk.img" is now a bootable image containing the Limine bootloader
and a multiboot(v1) compliant kernel.

### Automating The Process

It is now time to automate this process of image creation. But there has to be a couple of small changes.

First of all, in a script, the interactive mode isn't really a good fit. So instead of fdisk, parted is the
better option. Also instead of operating on the same output image, it makes more sense to me to work on a
temporary file and when everything is successful, it's feasible to just copy it. Pretty much everything else
is the same expect for some messages and variables.

```bash
# Define the ANSI escape sequence for red color and reset
RED='\033[0;31m'
GREEN='\033[0;32m'
RESET='\033[0m'

readonly SCRIPT_DIR=$(dirname "$0")
readonly BUILD_DIR="${SCRIPT_DIR}/../build"
readonly KERNEL_PATH="${BUILD_DIR}/kernel/kernel"
readonly IMG_FILE="${BUILD_DIR}/os.iso"
readonly LIMINE_DIR="${BUILD_DIR}/limine"
readonly LIMINE_CFG="${BUILD_DIR}/../boot/limine.cfg"
readonly TMP_IMG="/tmp/disk.iso"

if [ ! -e $KERNEL_PATH ]; then
  printf "${RED}Kernel file could not be found!${RESET}\n";
  exit 1;
fi


printf "${GREEN}Creating the image...${RESET}\n";
dd if=/dev/zero of=${TMP_IMG} bs=1M count=0 seek=64
if [ $? -ne 0 ]; then
  printf "${RED}Could not create the tmp image file!${RESET}\n";
  exit 1;
fi

printf "${GREEN}Preparing the image...${RESET}\n";
# Create a GPT partition table.
parted -s ${TMP_IMG} mklabel gpt
if [ $? -ne 0 ]; then
  printf "${RED}Could not create the partition table on tmp image file!${RESET}\n";
  exit 1;
fi

# Create an ESP partition that spans the whole disk.
parted -s ${TMP_IMG} mkpart ESP fat32 2048s 100%
if [ $? -ne 0 ]; then
  printf "${RED}Could not format the partition on tmp image file!${RESET}\n";
  exit 1;
fi

parted -s ${TMP_IMG} set 1 esp on
if [ $? -ne 0 ]; then
  printf "${RED}Could not set the partition as ESP on tmp image file!${RESET}\n";
  exit 1;
fi

printf "${GREEN}Preparing the limine...${RESET}\n";
if [ ! -d "${LIMINE_DIR}" ]; then
  git clone --branch=v4.x-branch-binary --depth=1 https://github.com/limine-bootloader/limine.git ${LIMINE_DIR}
  make -C "${LIMINE_DIR}"
fi

printf "${GREEN}Depolying limine on the image...${RESET}\n";
${LIMINE_DIR}/limine-deploy ${TMP_IMG}
if [ $? -ne 0 ]; then
  printf "${RED}Could not deploy limine on tmp image file!${RESET}\n";
  exit 1;
fi

printf "${GREEN}Setting up the loopback device...${RESET}\n";
readonly USED_LOOPBACK=$(sudo losetup -Pf --show ${TMP_IMG})

printf "${GREEN}Formatting the boot partition...${RESET}\n";
sudo mkfs.fat -F 32 ${USED_LOOPBACK}p1

printf "${GREEN}Mounting the boot partition...${RESET}\n";
mkdir -p img_mount
sudo mount ${USED_LOOPBACK}p1 img_mount

printf "${GREEN}Setting up files...${RESET}\n";
sudo mkdir -p img_mount/EFI/BOOT
sudo cp -v ${KERNEL_PATH} ${LIMINE_CFG} limine/limine.sys img_mount/
sudo cp -v limine/BOOTX64.EFI img_mount/EFI/BOOT/

# Sync system cache and unmount partition and loopback device.
sync
sudo umount img_mount
sudo losetup -d ${USED_LOOPBACK}

cp -v ${TMP_IMG} ${IMG_FILE}
printf "${GREEN}Done!${RESET}\n";
```

As it is now, it can be executable by a bash subprocess, like:

```
bash make_iso.sh
```

But it needs to be integrated with cmake, since that's how my build system works. To do this, there
needs to be a new custom target that will be execute this script every time the "kernel" target is updated. And
cmake needs to know about what it generates so that other targets can DEPEND on it.

```cmake
add_custom_target(limine-image ALL
    COMMAND "${CMAKE_COMMAND}" -E env "OS_BUILD_DIR=${CMAKE_BINARY_DIR}" "LIMINE_CFG=${CMAKE_CURRENT_SOURCE_DIR}/boot/limine.cfg" "${CMAKE_CURRENT_SOURCE_DIR}/meta/make_iso.sh"
    BYPRODUCTS "${CMAKE_BINARY_DIR}/limine_image.iso"
    USES_TERMINAL
    DEPENDS kernel
)
```

This target depends on the "kernel" target and creates a by product named "limine\_image.iso" by
executing the "make\_iso.sh" script.

As can be seen here, when executing the script a couple of environment variables also has been passed, so
that some parameters can be passed from cmake down to the script, this gives a bit more flexibility. Script has
also been updated to use these.

```bash
# Define the ANSI escape sequence for red color and reset
RED='\033[0;31m'
GREEN='\033[0;32m'
RESET='\033[0m'

readonly SCRIPT_DIR=$(dirname "$0")
readonly KERNEL_PATH="${OS_BUILD_DIR}/kernel/kernel"
readonly IMG_FILE="${OS_BUILD_DIR}/limine_image.iso"
readonly LIMINE_DIR="${OS_BUILD_DIR}/limine"
readonly TMP_IMG="/tmp/disk.iso"

if [ ! -e $KERNEL_PATH ]; then
  printf "${RED}Kernel file could not be found!${RESET}\n";
  exit 1;
fi


printf "${GREEN}Creating the image...${RESET}\n";
dd if=/dev/zero of=${TMP_IMG} bs=1M count=0 seek=64
if [ $? -ne 0 ]; then
  printf "${RED}Could not create the tmp image file!${RESET}\n";
  exit 1;
fi

printf "${GREEN}Preparing the image...${RESET}\n";
# Create a GPT partition table.
parted -s ${TMP_IMG} mklabel gpt
if [ $? -ne 0 ]; then
  printf "${RED}Could not create the partition table on tmp image file!${RESET}\n";
  exit 1;
fi

# Create an ESP partition that spans the whole disk.
parted -s ${TMP_IMG} mkpart ESP fat32 2048s 100%
if [ $? -ne 0 ]; then
  printf "${RED}Could not format the partition on tmp image file!${RESET}\n";
  exit 1;
fi

parted -s ${TMP_IMG} set 1 esp on
if [ $? -ne 0 ]; then
  printf "${RED}Could not set the partition as ESP on tmp image file!${RESET}\n";
  exit 1;
fi

printf "${GREEN}Preparing the limine...${RESET}\n";
if [ ! -d "${LIMINE_DIR}" ]; then
  git clone --branch=v4.x-branch-binary --depth=1 https://github.com/limine-bootloader/limine.git ${LIMINE_DIR}
  make -C "${LIMINE_DIR}"
fi

printf "${GREEN}Depolying limine on the image...${RESET}\n";
${LIMINE_DIR}/limine-deploy ${TMP_IMG}
if [ $? -ne 0 ]; then
  printf "${RED}Could not deploy limine on tmp image file!${RESET}\n";
  exit 1;
fi

printf "${GREEN}Setting up the loopback device...${RESET}\n";
readonly USED_LOOPBACK=$(sudo losetup -Pf --show ${TMP_IMG})

printf "${GREEN}Formatting the boot partition...${RESET}\n";
sudo mkfs.fat -F 32 ${USED_LOOPBACK}p1

printf "${GREEN}Mounting the boot partition...${RESET}\n";
mkdir -p img_mount
sudo mount ${USED_LOOPBACK}p1 img_mount

printf "${GREEN}Setting up files...${RESET}\n";
sudo mkdir -p img_mount/EFI/BOOT
sudo cp -v ${KERNEL_PATH} ${LIMINE_CFG} ${OS_BUILD_DIR}/limine/limine.sys img_mount/
sudo cp -v ${OS_BUILD_DIR}/limine/BOOTX64.EFI img_mount/EFI/BOOT/

# Sync system cache and unmount partition and loopback device.
sync
sudo umount img_mount
sudo losetup -d ${USED_LOOPBACK}

cp -v ${TMP_IMG} ${IMG_FILE}
printf "${GREEN}Done!${RESET}\n";
```

And this is the end. Now every time the build command is executed, for example "ninja", it will
call this script when there is a change to the kernel executable and this script will build
the image with the Limine bootloader in it.
