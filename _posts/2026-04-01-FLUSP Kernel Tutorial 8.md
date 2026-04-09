---
categories: [Dev, Kernel_Dev]
date: 2026-04-01
tags: [kernel, mac5856]
title: Tutorial 8 IIO Dummy module Experiment One: Play with iio_dummy
---

(available at <https://flusp.ime.usp.br/iio/experiment-one-iio-dummy/>)

Moving from Tutorial 7 to Tutorial 8 was like stepping out of a classroom and directly into a construction site. While the previous tutorial was all about reading code, this one was about making things work—and as a beginner, I hit quite a few walls. The goal was simple: enable, compile, and load the `iio_dummy` module. However, the reality involved permission issues, compiler mismatches, and the confusing dance between my host machine and the Virtual Machine (VM).

## 1. Enable the IIO dummy module via nconfig

My first task was enabling the IIO dummy module. I had to go back to Tutorial 3 to remember how to use `nconfig`. I ran `make -C "$IIO_TREE" menuconfig`, but immediately hit a permission wall, which I solved by adding `sudo`. Once configured, I double-checked everything by searching the `.config` file with `cat /home/lk_dev/iio/.config | grep "DUMMY"`. So far, so good.

The real trouble started during compilation. When I tried `sudo make M=drivers/iio/dummy`, the terminal exploded with errors. The compiler complained that it differed from the one used to build the kernel, and then I hit a fatal error: `asm/compiler.h: No such file or directory`. It felt like the tutorial was missing a step, but I found the fix: I needed to prepare the kernel headers first. Running `sudo make modules_prepare` before compiling the module finally allowed the process to finish.

## 2. Compile the IIO dummy module

Here is where I got really turned around. I accidentally ran `sudo make modules_install` and `sudo modprobe iio_dummy` on my host machine instead of inside the VM. I even asked ChatGPT, which confused me further by suggesting these commands only work in the kernel source directory on the host. I felt completely stuck until I realized that in this lab, we need to install the modules into the VM's filesystem without being logged into it. I had to use `guestmount` to mount the VM's image (as made in Tutorial 3):

- Mounting: I tried mounting it to the same directory as Tutorial 1, but got a "mountpoint is not empty" error. I solved this by creating a fresh directory: `mkdir ${VM_DIR}/arm64_new_rootfs`
- Installing: With the VM filesystem mounted, I ran `sudo --preserve-env make -C "${IIO_TREE}" INSTALL_MOD_PATH="${VM_DIR}/arm64_new_rootfs" modules_install`.
- Unmounting: Finally, `sudo guestunmount "${VM_DIR}/arm64_new_rootfs"`.

## 3. Load and unload iio_dummy module and 4. Inspect the /sys/bus/iio/*

Even after getting the modules into the VM, I faced connection issues with `kw ssh`. It couldn't reach `localhost:22`. I realized I needed to point it to the actual VM IP, so I ran `kw remote --add arm64 root@192.168.122.226 --set-default`.

Once inside the VM, I tried `modprobe iio_dummy`, but nothing appeared in `/sys/bus/iio/devices/`. I panicked for a second, then remembered: I had unloaded the module earlier! After running `sudo modprobe iio_dummy` again and creating a device via configfs with `sudo mkdir /mnt/iio_experiments/iio/devices/dummy/my_glorious_dummy_device`, the magic finally happened. The device appeared as `iio:device0`.


## 5. Modify iio_simple_dummy module to add channels for a 3-axis compass

The last step was to add a new channel for a Magnetometer (X-axis) to the code in `/home/lk_dev/iio/drivers/iio/dummy/iio_simple_dummy.c`. I added the following snippet for each axis (X, Y and Z):

```bash
{
    .type = IIO_MAGN,
    .modified = 1,
    .channel2 = IIO_MOD_X,
    .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
    .info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),
    .scan_index = DUMMY_MAGN_X,
    .scan_type = {
        .sign = 'u',
        .realbits = 16,
        .storagebits = 16,
        .shift = 0,
    },
},

```
After rebuilding and sending the modules again with `kw`, I verified the change. By checking `/sys/bus/iio/devices/iio:device0/`, I could finally see my new attributes: `in_magn_x_raw` and `in_magn_scale`. Reading the values with cat gave me actual data (like 78 for raw and 0.000002 for scale). Seeing those numbers on the screen made all the hours of troubleshooting and "reading four times" totally worth it.
