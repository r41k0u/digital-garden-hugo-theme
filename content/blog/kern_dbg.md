---
title: Interactive debugging of the Linux kernel on a Raspberry Pi 5
date: 2025-08-15
---

A while back, I had to debug the Broadcom FMAC WiFi driver in the linux kernel, which was causing a [bug](https://github.com/raspberrypi/linux/issues/6049) on the Raspberry Pi 5, 4 and some pinebooks as well (also documented in [Launchpad](https://bugs.launchpad.net/ubuntu/+source/linux-raspi/+bug/2063365) and [kernel lore](https://lore.kernel.org/all/d9c9336a-6314-4de9-aead-8b865bb30f05@gmx.net/)). Thi blog post is about how I set up interactive kernel debugging on a Raspberry Pi 5 with the [Raspberry Pi debug probe](https://www.raspberrypi.com/documentation/microcontrollers/debug-probe.html).

I also gave an LT talk about this at one of the Canonical sprints - which was recorded and can be viewed [here](../linux_kernel_pi5_debugging.mov)!

## Different ways to debug the linux kernel

First let's talk about the various ways we have to debug the linux kernel.

### dynamic_debug and the debugfs

Classic printline debugging! If your kernel has `CONFIG_DYNAMIC_DEBUG` enabled, you can use the `dynamic_debug` feature to enable/disable debug messages at runtime by writing into `<debugfs>/dynamic_debug/control`.

It can be used to enable messages from the prdbg API, which means the following functions:

```C
pr_debug()
dev_dbg()
print_hex_dump_debug()
print_hex_dump_bytes()
```

We can also print other metadata like the thread ID, module name, function, file and line number. We can enable these messages by specifying a line range, a file, a function, or a kernel module. Here is an example:

```bash
name@name-desktop:/sys/kernel$ cat debug/dynamic_debug/control | grep pisp
drivers/media/platform/raspberrypi/pisp_be/pisp_be.c:933 [pisp_be]pispbe_node_start_streaming =_ "Nodes streaming for this group now 0x%x\n"
drivers/media/platform/raspberrypi/pisp_be/pisp_be.c:931 [pisp_be]pispbe_node_start_streaming =_ "%s: for node %s (count %u)\n"
drivers/media/platform/raspberrypi/pisp_be/pisp_be.c:901 [pisp_be]pispbe_node_buffer_queue =_ "%s: for node %s\n"
drivers/media/platform/raspberrypi/pisp_be/pisp_be.c:716 [pisp_be]pispbe_isr =_ "Job counters not matching: done = %u, expected %u - started = %u, expected %u\n"
...
```

Suppose we want to enable the debug messages occuring between lines 900 and 950 in the `pisp_be` driver, so we must do the following:

```bash
name@name-desktop:/sys/kernel$ alias ddcmd='echo $* > /proc/dynamic_debug/control'
name@name-desktop:/sys/kernel$ ddcmd file pisp_be.c line 900-950 +pmf
name@name-desktop:/sys/kernel$ cat debug/dynamic_debug/control | grep pisp
drivers/media/platform/raspberrypi/pisp_be/pisp_be.c:933 [pisp_be]pispbe_node_start_streaming =pmf "Nodes streaming for this group now 0x%x\n"
drivers/media/platform/raspberrypi/pisp_be/pisp_be.c:931 [pisp_be]pispbe_node_start_streaming =pmf "%s: for node %s (count %u)\n"
drivers/media/platform/raspberrypi/pisp_be/pisp_be.c:901 [pisp_be]pispbe_node_buffer_queue =pmf "%s: for node %s\n"
drivers/media/platform/raspberrypi/pisp_be/pisp_be.c:716 [pisp_be]pispbe_isr =_ "Job counters not matching: done = %u, expected %u - started = %u, expected %u\n"
...
```

You can see that the `pmf` flags have been added to the first 3 entries, which means that these debug sites have been enabled, and will also print the module name and the function name (`m and f`) to the kernel ringbuffer. You can read more in the full [documentation of <code>dynamic_debug</code>](https://www.kernel.org/doc/html/v6.16/admin-guide/dynamic-debug-howto.html).

This is a very powerful feature, as you don't have to recompile the kernel and you can enable/disable debug messages at runtime. However, it is not interactive, and you have to know the file and line number of the debug site you want to enable. Additionally, you can only enable debug messages already present in the kernel source code, so you cannot add new debug messages on the fly.

I wanted a more interactive way to debug the kernel, much like you would do with a debugger like gdb, so I looked into other options.

### GDB with QEMU

You can use the gdbstub in QEMU to debug anything running in it, including the linux kernel. You can read more about it in the [GDB usage](https://qemu-project.gitlab.io/qemu/system/gdb.html) section of the QEMU documentation. This is convenient, as you don't need only 1 device and you can debug issues of other architectures as well, but this didn't work for me. I had to debug the issue on a Raspberry Pi 5, but [QEMU does not emulate the Raspberry Pi 5](https://www.qemu.org/docs/master/system/arm/raspi.html) yet.

### KGDB?

The linux kernel has a gdbserver stub built-in, called KGDB. It allows you to connect to gdb using a serial or ethernet connection. You can also use KDB, which is a console-based debugger that allows you to inspect the kernel state and execute commands - and it works on a single device.

For using KGDB, you need to compile the kernel with the KGDB options. Then you need to specify the `kgdboc` kernel command line option to enable the KGDB over console. You can also use the `kgdbwait` option to make the kernel wait for a gdb connection before booting. You can find more information on this in the [KGDB documentation](https://www.kernel.org/doc/html/v6.16/process/debugging/kgdb.html).

I am glossing over this section, as I didn't use KGDB for debugging this issue, but this will suffice for most people if you intend to just debug the **Linux** kernel.
My reasons for not using it:

- I was using the Raspberry Pi Debug Probe as the serial adapter, but turns out that I can also use it's SWD interface (explained in the next section).
- I had to setup an interrupt proxy, as I did not have 2 keyboards to issue the SysRq-G interrupt to halt the target machine.
- This is only applicable for the Linux kernel. I wanted a way that could be used to debug any kernel (for future use-cases/projects).

### GDB over SWD with the Raspberry Pi Debug Probe

I had a Pi debug probe which I used as a serial adapter to check the early boot messages of the Raspberry Pi 5. While reading [it's docs](https://www.raspberrypi.com/documentation/microcontrollers/debug-probe.html), I noticed that it supports SWD and the CMSIS-DAP standard. The [UART port](https://datasheets.raspberrypi.com/debug/debug-connector-specification.pdf) on the Raspberry Pi 5 also supports SWD and cJTAG, so is there a way I could use this arrangement to debug the kernel? Not just the linux kernel, but any custom kernel as well?

The [CMSIS-DAP standard](https://developer.arm.com/documentation/101451/0100/About-CMSIS-DAP) defines a firmware interface to the [CoreSight debug and trace]() units present on ARM Cortex CPUs. So you can debug anything running on an ARM Cortex CPU using this (including the linux kernel).

![CMSIS-DAP](https://arm-software.github.io/CMSIS_5/DAP/html/CMSIS_DAP_INTERFACE.png)

We can use [OpenOCD](https://openocd.org/) to act as a translation layer betwen the gdb frontend and the CMSIS-DAP backend, as it supports the Raspberry Pi Debug Probe.
This is the scaled down representation of our debug chain:
![Final debug chain](https://www.allaboutcircuits.com/uploads/articles/OpenOCD_Diag.png)

## Setup

### Compiling the kernel with KGDB and GDB scripts

First, we will compile the kernel with the necessary options. I compiled this for using KGDB as well.
You can largely follow the [Raspberry Pi kernel compilation guide](https://www.raspberrypi.com/documentation/computers/linux_kernel.html) to compile the kernel from any source (not just Raspberry Pi's fork). Here are the options you need to enable in the kernel configuration (during or after `make menuconfig`):

```
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
CONFIG_GDB_SCRIPTS=y
...
```

This is not an exhaustive list, you can find more options relevant to your use-case by searching around the web. I find [kernelconfig.io](https://www.kernelconfig.io/index.html) to be a handy tool for this.

I also find it useful to set a `CONFIG_LOCALVERSION` with a `-dbg` suffix, so I can identify the kernel and dtb files easily and not confuse them with other builds.

Assuming you build for arm64:

```bash
name@name-desktop:~/linux$ make -j6 Image.gz modules dtbs
name@name-desktop:~/linux$ make modules_install
```

Also check if it was installed correctly by looking at the new directory under `/lib/modules/`

### Updating the boot artefacts

**You have to be a bit careful here, as messing this up can lead to boot failures, so make sure to keep backups of everything you replace.**

- Copy the `System.map` and `.config` from your kernel source to `/boot` with the kernel version string (`$VERSION`) suffixed, like `/boot/System.map-$VERSION` and `/boot/config-$VERSION`.
- Make `boot/dtbs/$VERSION/` and copy your Raspberry Pi's device tree blob into it. `bcm2712-rpi-5-b.dtb` is the dtb for the Raspberry Pi 5, which can be found in `arch/arm64/boot/dts/broadcom/` in your kernel source.
- Make a symlink `/boot/dtb-$VERSION` which points to `boot/dtbs/$VERSION/./<your-dtb>.dtb` (here `<your-dtb>` is `bcm2712-rpi-5-b`).
- Update the `/boot/dtb` symlink to point to `boot/dtbs/$VERSION/./<your-dtb>.dtb`.
- Take backups of `/boot/vmlinuz*` (* is th wildcard here, so any current or older vmlinuz). Don't update the vmlinux symlink yet, make `vmlinuz.old*` symlinks to point to older kernel images.
- Copy vmlinuz from your kernel source to `/boot` as `/boot/vmlinuz-$VERSION`. Update the `/boot/vmlinuz` symlink to point to `/boot/vmlinuz-$VERSION`.
- Next we will generate the initramfs. Copy `<your-dtb>.dtb` to `/etc/flash-kernel/dtbs`
- Make sure that you use `MODULES=dep` in `etc/initramfs-tools/initramfs.conf` (or `/etc/initramfs-tools/conf.d/` if it overrides the original config) to generate a smaller initramfs which can fit in the Raspberry Pi 5's boot partition (which is 512MB by default).
- Generate the initramfs using `update-initramfs -c -k $VERSION`.
- Update the `/boot/initrd.img` symlink to point to the newly generated initramfs, and make `initrd.img.old` symlink to point to the previous initramfs.
- Check if `/boot/firmware/vmlinuz` is the same as `/boot/vmlinuz-$VERSION`. If not, take backups of existing `/boot/firmware/vmlinuz*` and copy the new vmlinuz into `/boot/firmware`.
- Check if `/boot/firmware/initrd.img` is the same as `/boot/initrd.img-$VERSION`. If not, take backups of existing `/boot/firmware/initrd.img*` and copy the new `initrd.img` into `/boot/firmware`.
- Add `enable_uart=1` and `enable_jtag_gpio=1` to `/boot/config.txt`.
- Add `nokaslr rodata=off maxcpus=1` to `/boot/cmdline.txt`.
-   - `nokaslr` disables kernel ASLR.
-   - `rodata=off` disables read-only data sections, which is useful for debugging.
-   - `maxcpus=1` limits to only 1 CPU core, which avoids the complexity of threading issues while debugging.

After this, you must reboot your Raspberry Pi 5. If it boots successfully and `uname -r` shows the new kernel version, you are good to go to the next step.

### Setting up OpenOCD

Next, we setup OpenOCD on another computer which will act as the monitoring computer.

First, copy the compiled kernel source to your monitoring computer. Install `OpenOCD` and `gdb-multiarch`. I use Ubuntu, so I installed them from `apt`.

Take the Raspberry Pi Debug probe and connect it's **D** port to the UART port of the Raspberry Pi 5. Connect it to the monitoring computer using USB.

Next, we need the config files for the CMSIS-DAP interface, and the Raspberry Pi 5 target device. While `cmsis-dap.cfg` is bundled with OpenOCD, the target config for BCM2712 (the SoC for Raspberry Pi 5) can be found in [this forum post by user trejan](https://forums.raspberrypi.com/viewtopic.php?p=2172522#p2172235).

Copy the configs to a directory, and reboot your target (in this case, the Raspberry Pi 5).

Now you can start OpenOCD:

```bash
name@name-desktop:~/openocd$ openocd -f cmsis-dap.cfg -f bcm2712.cfg
```

This should spin up gdbservers for each CPU core. The output should be something like this:
![OpenOCD output](https://i.postimg.cc/zvDwbxbR/openocd.png)

### Connecting GDB to OpenOCD

Use `gdb-multiarch` (if your monitoring computer is not an ARM64 machine, else you can use `gdb` directly) to connect to the OpenOCD gdbserver for CPU0:

```bash
name@name-desktop:~/openocd$ gdb-multiarch vmlinux
...
(gdb) tar extended-remote localhost:3333
```

This should have halted the CPU of your target machine (no blinking cursor), and you'll see that it stopped somewhere in `kernel/idle.c`
![OpenOCD gdb output](https://i.postimg.cc/xCGnnxjw/image.png)

You can now halt the execution of the CPU by issuing an interrupt, and then inspect the backtrace, the registers, memory etc.
But there are 2 main catches here which you'll see after playing around a bit:

- You notice the lack of symbols in the backtraces.
- You notice that you cannot set breakpoints in kernel modules.

![No symbols](https://i.postimg.cc/85b35q8V/image.png)

Clearly, we need to load the symbols of the kernel modules to make this more productive.

### Adding symbols manually

To add symbols, you need to load the symbol files in gdb at the address of the `.text` section of the relevant kernel module.

To find the address of the `.text` section, I did the following on the target machine:

```bash
name@name-desktop:~$ cat /sys/module/<kernel-module-name>/sections/.text
```

The `<kernel-module-name>` is the name of the kernel module you want to debug. For me it was `brcmfmac`. Then you, add the corresponding symbol file on the monitoring machine at the right address:

```bash
(gdb) add-symbol-file <path-to-kernel-module>.ko <address in .text>
```

Doing this for some kernel modules, I realised that there should be a better way of doing this, as this seemed extremely inefficient. That's when I stumbled upon `lx-scripts` (after a bit of RTFM).

### Using lx-scripts

For some reason, even after enabling `CONFIG_GDB_SCRIPTS`, the lx-scripts did not compile for me. I manually compiled them (they're present at `scripts/gdb/*` in the kernel source) and added them to the `~/.config/gdb/gdbinit` as a safe-path:

```bash
add-auto-load-safe-path <kernel-source>/scripts/gdb/vmlinux-gdb.py
```

Now, you can reload gdb, connect to the OpenOCD gdbserver, and run `lx-symbols`:

```bash
(gdb) lx-symbols
```

![lx-symbols output](https://i.postimg.cc/mZXX53BV/image.png)

This automates the process of loading the symbols for all the kernel modules, which allows you to interactively debug them as well! This also adds the symbols for most of the kernel stack, so you can now go through different frames and inspect the memory.

![glorious symbols](https://i.postimg.cc/2jKGwz7K/image.png)

Now you can debug the kernel running on a real hardware just like you would do a binary running on your computer.

## Final thoughts

This setup is quite nice, as I can debug kernel bugs specific to the Raspberry Pi 5 on real hardware now, and also debug custom kernels, which helps a lot if you are developing one. The setup can feel overwheming, but it is quite simple once you figure it out.

I was also surprised to see that such a tutorial for the Raspberry Pi debug probe was not available. Considering that the Pi ecosystem is fairly educational, I thought that such a tutorial would help many people who want to debug the linux kernel interactively on real hardware.

Thank you for reading this post! If you have any questions/suggestions, feel free to mail them to me!
