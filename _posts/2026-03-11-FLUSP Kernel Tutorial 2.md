---
categories: [Dev, Kernel_Dev]
date: 2026-03-11
tags: [kernel, mac5856]
title: Tutorial 2 Building and booting a custom Linux kernel for ARM using kw
---

(available at <https://flusp.ime.usp.br/kernel/build-linux-for-arm-kw/>)

As a beginner, I felt more confident starting this tutorial because I
was already somewhat familiar with the environment and with some of the
tools used in the previous tutorial.

**kworkflow**, or simply `kw`, is a *Free/Libre and Open Source Software
(FLOSS) Developer Automation Workflow System (DAWS)* whose mission is to
reduce configuration and environment overhead in Linux development. The
`kw` project helps kernel developers with everyday tasks, such as
**automating and simplifying the process of compiling and deploying
custom Linux kernels from source code**.

Right at the beginning I encountered an error when trying to check the
version information of `kw`:

``` bash
(LK-DEV) lais@gausss:/home/lk_dev/kw$ kw --version
/home/lais/.local/bin/kw: line 75: /usr/share/kw/lib/kw_include.sh: No such file or directory
/home/lais/.local/bin/kw: line 77: include: command not found
/home/lais/.local/bin/kw: line 109: signal_manager: command not found
/home/lais/.local/bin/kw: line 109: warning: command not found
/home/lais/.local/bin/kw: line 294: include: command not found
/home/lais/.local/bin/kw: line 296: kworkflow_version: command not found
```

To solve this problem I had to directly invoke the executable inside the
project directory using: `$KW_DIR/kw [command]`. In other words, I
needed to use this format whenever I executed any `kw` command. I ended
up spending quite some time on this step because I had to reinstall and
redo the installation process several times before understanding what
was happening.

During step 4 of the tutorial, when attempting to fetch the module list from the virtual machine using:

```bash
$KW_DIR/kw ssh --get '~/vm_mod_list'
```
I encountered the following error:

```bash
(LK-DEV) root@gauss:/home/lk_dev/iio# $KW_DIR/kw ssh --get '~/vm_mod_list'
=========================================================
[INFO]: Running kw using repository executable.

KW environment variables are set to the following:

    KW_CACHE_DIR=/root/.cache/kw
    KW_DATA_DIR=/root/.local/share/kw
    KW_DB_DIR=/home/lk_dev/kw/database
    KW_DOC_DIR=/home/lk_dev/kw/documentation
    KW_ETC_DIR=/home/lk_dev/kw/etc
    KW_LIB_DIR=/home/lk_dev/kw/src
    KW_MAN_DIR=/home/lk_dev/kw/documentation/man
    KW_PLUGINS_DIR=/home/lk_dev/kw/src/plugins
    KW_SRC_LIB_DIR=/home/lk_dev/kw/src/lib
    KW_SOUND_DIR=/home/lk_dev/kw/sound

=========================================================
bash: line 1: rsync: command not found
rsync: connection unexpectedly closed (0 bytes received so far) [Receiver]
rsync error: error in rsync protocol data stream (code 12) at io.c(232) [Receiver=3.2.7]
An error occurred while uploading the file(s). rsync return code: 12
```
This happens because `kw ssh --get` relies on `rsync` over SSH to transfer files from the virtual machine to the host. Since rsync was not installed on the VM, the command failed. To fix this issue, I installed rsync inside the virtual machine:

``` bash
apt update
apt install rsync -y
```
After installing `rsync`, the command worked correctly and I was able to proceed with generating the optimized kernel configuration.

Moreover, in step 4, the tutorial shows how to check some information about the
kernel compilation, including how many modules will be compiled, using:

``` bash
$KW_DIR/kw build --info
```

The result was the following:

``` bash
(LK-DEV) lais@gausss:/home/lk_dev/iio$ $KW_DIR/kw build --info
=========================================================
[INFO]: Running kw using repository executable.

KW environment variables are set to the following:

    KW_CACHE_DIR=/home/lais/.cache/kw
    KW_DATA_DIR=/home/lais/.local/share/kw
    KW_DB_DIR=/home/lk_dev/kw/database
    KW_DOC_DIR=/home/lk_dev/kw/documentation
    KW_ETC_DIR=/home/lk_dev/kw/etc
    KW_LIB_DIR=/home/lk_dev/kw/src
    KW_MAN_DIR=/home/lk_dev/kw/documentation/man
    KW_PLUGINS_DIR=/home/lk_dev/kw/src/plugins
    KW_SRC_LIB_DIR=/home/lk_dev/kw/src/lib
    KW_SOUND_DIR=/home/lk_dev/kw/sound

=========================================================
Kernel source information
  Name: 7.0.0-rc1-g9f60b8e91fd8
  Version: 7.0.0-rc1
  Total modules to be compiled: 9
```

After that I used the command:

``` bash
$KW_DIR/kw build
```

This command compiles the Linux kernel from the source code considering
the local configuration. The process took a few minutes to complete.

`Execution time: 00:14:49`

In step 6, the tutorial suggests checking whether the system is actually
running the custom kernel that we just compiled. To do this we use the
command:

``` bash
uname --kernel-release
```

The expected result should have a format similar to:

``` bash
root@localhost:~# uname --kernel-release
7.0.0-rc1VM-laistar+
```

This confirms that the VM is running the kernel that was compiled during
the tutorial.
