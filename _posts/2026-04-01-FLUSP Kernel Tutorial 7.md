---
categories: [Dev, Kernel_Dev]
date: 2026-04-01
tags: [kernel, mac5856]
title: Tutorial 7 The iio_simple_dummy Anatomy
---

(available at <https://flusp.ime.usp.br/iio/iio-dummy-anatomy/>)

After diving into the theoretical aspects of patches in previous tutorials, I finally reached Tutorial 7. I’ll be honest: as a beginner, I had to read this four times and search for several concepts to not feel completely lost. My first big question was: what exactly is the IIO subsystem? I found out that the Industrial I/O (IIO) subsystem is meant to support devices performing analog-to-digital (ADC) or digital-to-analog (DAC) conversions. It fills the gap between `hwmon` (used for low sample rate sensors like fans) and the input subsystem (focused on human interaction like keyboards). Another key concept I had to grasp was the buffer, which is essentially a memory region used to store data temporarily while it moves from one place to another.

The core of this setup is the `iio_chan_spec` structure. It is massively configurable, which adds a lot of flexibility because the same struct can configure different sets of devices. In the code, each element in the array describes a channel. For example, the `.type` field specifies what is being measured, like `IIO_VOLTAGE`. I also learned about the `.indexed` field; when set to 1, it means the channel has a numerical index, defined in the .channel field. This is how we distinguish multiple data channels of the same type. The `iio_dummy` uses this to create multiple channels and simulate a variety of sensors.

I noticed that these channels play a very important role in how information is shared. The .`info_mask_separate` field is used to register attributes like `IIO_CHAN_INFO_RAW` (unscaled measurements), `IIO_CHAN_INFO_OFFSET` (bias removal), and `IIO_CHAN_INFO_SCALE` (multiplier for conversion to standard units). There is also the .`info_mask_shared_by_type`, which represents attributes shared by channels of the same type, meaning other channels can access that information. This structure determines what userspace actually sees and how it calculates the final values.

The most confusing part for me was the relationship between buffers and the fields `.scan_index` and `.scan_type`. If `.scan_index` is -1, the channel doesn't support buffered capture. But if it's a unique positive number, it implements a buffer, and that index organizes the order of channels inside the buffer. The `.scan_type` then describes that data. There are also configurations for events using `.event_spec`, but those only work if `CONFIG_IIO_SIMPLE_DUMMY_EVENTS` is enabled. Since I felt quite confused during this tutorial, I decided to move on to Tutorial 8 to keep the momentum, keeping this anatomy open as a reference whenever I got stuck.
