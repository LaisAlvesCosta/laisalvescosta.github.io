---
categories: [Dev, Kernel_Dev]
date: 2026-03-18
tags: [kernel, mac5856]
title: Tutorial 4 Introduction to Linux kernel Character Device Drivers
---

(available at <https://flusp.ime.usp.br/kernel/char-drivers-intro/>)

Following my progress in the MAC5856 course, I have completed Tutorial 4, which introduces **Character Device Drivers**. Unlike the previous tutorial, this one was quite straightforward, and all steps were successful without major hurdles.

Character devices are abstractions in Linux for hardware that handles sequential data streams, typically in small transfers like single bytes. They are represented as **device nodes** (special files) in the file system, such as `/dev/ttyS0` or `/dev/keyboard`.

A key concept learned was the use of **Major and Minor numbers**:
*   The Major number identifies the specific driver associated with the device.
*   The Minor number is used by the driver to distinguish between different physical devices or modes of operation.

During the second step, I needed to identify the kernel image files. I used the following command to explore the VM image:
`virt-ls --add "${VM_DIR}/arm64_img.qcow2" --mount /dev/sda2 /boot`

Inside the VM, I verified the metadata of the kernel image:
```bash
root@localhost:~# stat /boot/vmlinuz-6.1.0-44-arm64
  File: /boot/vmlinuz-6.1.0-44-arm64
  Size: 32888768      Blocks: 64240      IO Block: 4096   regular file
Device: 254,2    Inode: 16033       Links: 1
```
The `stat` output confirms that the device responsible for this file has **major number 254** and **minor number 2** (which corresponds to `/dev/sda2`).

The driver uses a `struct file_operations` to map system calls like `open`, `read`, and `write` to specific functions in the kernel code. Because kernel and user space memories are separate, the driver must use functions like `copy_to_user()` and `copy_from_user()` to move data safely.

After loading the module, I checked its major number:
```bash
root@localhost:~# cat /proc/devices | grep simp
238 simple_char
```
Using `stat` on the device node `simple_char_node`, I verified the **Device type** field, which showed **238,0** (Major 238, Minor 0).

On my host machine, I wrote and cross-compiled two small programs to interact with the driver:
1.  **`read_prog.c`**: To read data from the character device.
2.  **`write_prog.c`**: To send data to the device buffer.

I compiled them for the VM architecture using the cross-compiler:
`aarch64-linux-gnu-gcc read_prog.c -o read_prog`

After transferring them via `scp` to the VM, the tests were successful:
*   **Read:** Successfully retrieved the message: *"This is data from simple_char buffer."*
*   **Write:** Successfully wrote 256 bytes to the device buffer.


This tutorial provided a clear view of how user-space applications communicate with kernel-space drivers through the file system abstraction. Understanding how Major/Minor numbers route these requests to the correct `cdev` (character device) structure is a fundamental skill for any kernel developer.