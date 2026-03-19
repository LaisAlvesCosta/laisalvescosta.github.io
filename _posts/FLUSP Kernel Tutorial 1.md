---
categories: [Dev, Kernel_Dev]
date: 2026-03-04
tags: [kernel, mac5856] # TAG names should always be lowercase
title: Tutorial 1 Setting up a test environment for Linux Kernel Dev using QEMU and libvirt
---

(available at <https://flusp.ime.usp.br/kernel/qemu-libvirt-setup/>)

I am a beginner in the field of Computer Science and I am still getting
used to Linux and some technical terms in the area, such as SSH and I/O
(IIO). First, I chose to set up **Dual Boot**, because I was afraid that
I might not be able to use my computer if something went wrong. For the
same reason, I am using **Ubuntu**, which has a more user‑friendly
interface. I went through this tutorial **three times** to better
consolidate the content.

The possibility of creating **virtual machines (VMs)** on my own
computer is something I have always found brilliant. I have often
thought about the dangers of viruses or tests that could end up damaging
or corrupting the main machine. However, before this tutorial, I would
not have known where to start to make this idea a reality. Beginning
this process within the context of **Linux Kernel development** was very
motivating.

In the tutorial, we use **QEMU** to run virtual machines. It provides a
virtual model of an entire machine (CPU, memory, and emulated devices)
that runs on top of the real operating system as if it were an
application. Meanwhile, **libvirt** simplifies the management of these
VMs and enables automation. Together, they provide a safe and efficient
way to test kernel modifications without risking the stability of the
main system.

To install the dependencies required for the tutorial, I used the
following command (since Ubuntu is based on Debian):

``` bash
# Debian-based distros
sudo apt update && sudo apt install qemu-system libvirt-daemon-system virtinst libguestfs-tools wget
```

I found the manipulation of variables that store the paths of essential
directories in the `activate.sh` file interesting, because they greatly
simplify the installation process and the organization of the
environment.

A small issue occurred when downloading the **Debian nocloud image**,
which was not found directly. I had to manually access:

<https://cdimage.debian.org/cdimage/cloud/bookworm/daily/>

and look for the folder corresponding to the date when I was following
the tutorial. After that, I found the indicated file, copied the URL,
and replaced it in the tutorial command. The image size is **389MB**,
and the tutorial mentions that **3GB is sufficient to install some
kernel modules** (which will be covered later).

When calling the function `launch_vm_qemu()` I had difficulty with
**permissions**, so I had to run it using `sudo`.

To **shut down the VM**, the command depends on the distribution being
used, but in my case I used:

``` bash
poweroff
```

`virsh` will be the tool used to manage VMs with libvirt. Some
installations do not allow non‑root users to run virsh by default. In
this case, it is possible to enable this by defining the default libvirt
URI: `export LIBVIRT_DEFAULT_URI=qemu:///system` and adding
`--connect qemu:///system` to the virsh commands in `activate.sh`. I did
this, but I must admit that I felt a bit confused at this part.

Another subtle detail that confused me slightly was the function
`create_vm_virsh`, because it can only be called once, since it creates
a VM named `arm64`. To open an already created VM, I used:

``` bash
sudo virsh start --console arm64
```

To exit the console I used: **Ctrl + \]**, and to reconnect to the
console of a running VM:

``` bash
sudo virsh console arm64
```

Before step 3, I needed to install the ssh package. In addition, to
discover the VM's IP address, I had to open two terminals: one connected
to the VM and another to the host. Only then was I able to find the
address: **192.168.122.231/24**. Since the name was not recognized in
the ssh command, I removed the **/24** and used:

``` bash
ssh root@192.168.122.231
```

In the end, I was able to access the virtual machine via SSH, as
expected in this tutorial.
