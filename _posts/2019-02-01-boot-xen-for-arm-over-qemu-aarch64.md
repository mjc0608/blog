---
layout: post
title: Boot Xen for ARM over QEMU-AARCH64
---

#### Reference

1. https://wiki.xenproject.org/wiki/Xen\_ARM\_with\_Virtualization\_Extensions/qemu-system-aarch64
2. https://wiki.linaro.org/LEG/UEFIforQEMU

### Step 0: Pre-request

Install the cross-compile toolchain for ARM. e.g. aarch64-linux-gnu-gcc.

### Step 1: Compile Linux, Xen, and QEMU

For Linux:

```bash
$ make ARCH=arm64 CROSS_COMPILE=/usr/bin/aarch64-linux-gnu- defconfig
$ make ARCH=arm64 CROSS_COMPILE=/usr/bin/aarch64-linux-gnu- -j8
```

For Xen:

```bash
$ make dist-xen XEN_TARGET_ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j8
```

For QEMU:

```bash
$ ./configure --target-list=aarch64-softmmu --enable-sdl --prefix=/usr/local
$ make -j8
$ sudo make install
```

### Step 2: Download and Manipulate Ubuntu Image

Download the image at https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-arm64-uefi1.img. Then we will boot the image for several times.

#### First Boot

In the first boot, we delete unnecessary packages from the image, and copy some files to the image. Use the following command to boot the image (remember to change the paths of kernel and drive to your file locations):

```bash
$ qemu-system-aarch64 \
   -machine virt,gic_version=3 -machine virtualization=true \
   -cpu cortex-a57 -machine type=virt -nographic \
   -smp 4 -m 4096 \
   -kernel /path/to/linux.git/arch/arm64/boot/Image.gz \
   --append "console=ttyAMA0 root=/dev/vda1 init=/bin/sh" \
   -netdev user,id=hostnet0,hostfwd=tcp::2222-:22 \
   -device virtio-net-device,netdev=hostnet0,mac=MAC_ADDRESS \
   -drive if=none,file=/path/to/xenial-server-cloudimg-arm64-uefi1.img,id=hd0 \
   -device virtio-blk-device,drive=hd0
```

Then, after booted, enter the following commands to remount the disk, reset root password, remove the (annoying) cloud-image packages, and open ssh server. (The sudo in the third line is necessary for a reason I cannot understand.)

```
$ mount -o remount,rw /dev/vda1 /
$ passwd root
$ sudo apt-get purge cloud-init cloud-guest-utils snapd
$ systemctl enable --now ssh
```

Also, remember to do one of the following:

1. edit `/etc/ssh/sshd_config` to allow root to login with password (`PermitRootLogin` & `PasswordAuthentication`)
2. copy your ssh public key to `/root/.ssh/authorized_keys`

Copy the linux kernel and initrd (both provided by ubuntu in the boot dir) to a place that xen can access during boot time. (We need to do this currently because I haven’t find a way to generate arm initrd on x86 platform.)

```bash
$ mount /dev/vda15 /boot/efi
$ mkdir /boot/efi/EFI/XEN
$ cp /boot/vmlinuz-4.4.0-141-generic /boot/efi/EFI/XEN
$ cp /boot/initrd.img-4.4.0-141-generic /boot/efi/EFI/XEN
$ umount /boot/efi
```

Edit `gurb.cfg`. Modify all lines like the following line

```
search --no-floppy --fs-uuid --set=root
```
to this:
```
search --no-floppy --label --set=root cloudimg-rootfs
```
Delete all quiet and splash option in the command linux.

Try to kill the QEMU process (since the poweroff command does not work).

#### Copy Xen to the Image

We will use qemu-nbd to copy Xen to the image.

```bash
$ sudo modprobe nbd
$ sudo qemu-nbd -c /dev/nbd0 /path/to/xenial-server-cloudimg-arm64-uefi1.img
$ sudo mount /dev/nbd0p15 /mnt
$ cd /mnt/EFI/XEN
$ sudo cp /path/to/xen.git/xen/xen xen.efi
```

Then, put the following configure in the XEN directory as `xen.cfg`:

```bash
options=console=dtuart noreboot dom0_mem=512M
kernel=vmlinuz-4.4.0-141-generic root=/dev/vda1 rw console=hvc0
ramdisk=initrd.img-4.4.0-141-generic
dtb=virt-gicv3.dtb
```

Also put `virt-gicv3.dtb` under the same directory. You may generate this file via:

```bash
$ qemu-system-aarch64 \
   -machine virt,gic_version=3 \
   -machine virtualization=true \
   -cpu cortex-a57 -machine type=virt \
   -smp 4 -m 4096 -display none \
   -machine dumpdtb=virt-gicv3.dtb
```

#### Second Boot

In the second boot, we configure the BIOS/UEFI.

Firstly, download the UEFI image at http://snapshots.linaro.org/components/kernel/leg-virt-tianocore-edk2-upstream/latest/QEMU-AARCH64/RELEASE\_GCC5/QEMU\_EFI.fd.

Create flash images (`flash0.img` and `flash1.img`). `flash0.img` contains the original UEFI, and `flash1.img` stores the configuration of UEFI.

```bash
$ cat QEMU_EFI.fd /dev/zero | dd iflag=fullblock bs=1M count=64 of=flash0.img
$ dd if=/dev/zero of=flash1.img bs=1M count=64
```
Then, boot qemu with the following command:

```bash
$ qemu-system-aarch64 \
   -machine virt,gic_version=3 -machine virtualization=true \
   -cpu cortex-a57 -machine type=virt\
   -smp 4 -m 4096 \
   -pflash bios/flash0.img -pflash bios/flash1.img \
   -hda img/xenial-server-cloudimg-arm64-uefi1.img \
   -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::50022-:22 \
   -display sdl -device virtio-gpu-pci -k en-us \
   -device usb-ehci -device usb-kbd -device usb-mouse -usb \
   -serial mon:stdio
```
In the graphic window, enter UEFI setting by press ESC. Then create a new boot entry of Xen and set it as the default one. Reset the machine, and then you should enter Xen by default.

### Step 3: Options

Add an NVMe device by adding the following line in qemu command:

```
-drive file=img/nvme.img,if=none,id=drv0 -device nvme,drive=drv0,serial=foo
```
Add a emulated fat32 device:
```
-drive file=fat:rw:your_dir_path
```
