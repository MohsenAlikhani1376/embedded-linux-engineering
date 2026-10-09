# Embedded Linux Learning Notes — QEMU ARM64 Environment

## 1. Project Goal

Build and run a minimal ARM64 Linux system using QEMU before moving to the real target hardware, Orange Pi 3 LTS.

Our environment consists of:

- **QEMU:** Emulates a virtual ARM64 machine.
- **Linux kernel:** Manages CPU, memory, devices, processes, and filesystems.
- **Cross-compiler:** Compiles programs for ARM64 on our x86-64 Ubuntu PC.
- **BusyBox:** Provides basic Linux commands and utilities.
- **Root filesystem (rootfs):** Contains the programs, configuration files, and directories needed by Linux userspace.
- **Initramfs:** An archive containing the root filesystem, loaded into RAM during boot.
- **BusyBox init:** Starts the userspace initialization process and reads our startup configuration.

### Important limitation

We use QEMU's generic `virt` machine with a Cortex-A53 CPU. This is useful for learning ARM64 Linux, but it does **not** emulate the Orange Pi 3 LTS's Allwinner H6 hardware.

---

## 2. Development Environment

### Host operating system

- Ubuntu 20.04.6 LTS running on an x86-64 PC.
- QEMU system emulator for AArch64.
- Linux kernel source code.
- BusyBox source code.
- AArch64 cross-compilation toolchain.

### Project directory structure

Our project is located at:

```bash
~/Desktop/embedded_linux/qemu/Orangepi3-qemu
```

The important files and directories are:

```text
Orangepi3-qemu/
├── linux/
│   └── arch/arm64/boot/Image
├── busybox/
├── rootfs/
│   ├── bin/
│   │   ├── busybox
│   │   └── sh -> busybox
│   ├── dev/
│   ├── etc/
│   │   ├── inittab
│   │   └── init.d/
│   │       └── rcS
│   ├── proc/
│   ├── sbin/
│   │   └── init -> ../bin/busybox
│   ├── sys/
│   ├── tmp/
│   └── linuxrc -> bin/busybox
└── rootfs.cpio.gz
```

**What each component means:**

- `linux/`: Linux kernel source and build output.
- `linux/arch/arm64/boot/Image`: The compiled ARM64 kernel.
- `busybox/`: BusyBox source code and build configuration.
- `rootfs/`: The root filesystem directory tree that we assembled.
- `rootfs.cpio.gz`: The compressed initramfs archive loaded by QEMU.

The `Image` file is the kernel; `rootfs.cpio.gz` contains the initial userspace environment. They are separate artifacts.

---

## 3. Step 1 — Configure the Cross-Compilation Environment

Our cross-compiler is installed at:

```text
/home/mohsen/toolchains/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
```

The compiler executable is:

```text
aarch64-none-linux-gnu-gcc
```

We configure the environment with:

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-none-linux-gnu-
```

**Explanation:**

- `ARCH=arm64` tells the Linux kernel build system which CPU architecture to target.
- `CROSS_COMPILE` provides the prefix for the cross-compilation tools.
- The trailing hyphen is intentional: the build system appends tool names such as `gcc`, `ld`, and `ar`.

For example, the compiler command becomes:

```text
aarch64-none-linux-gnu-gcc
```

The compiler runs on our x86-64 PC but generates ARM64 executables.

**Important:** These `export` commands configure only the current terminal session. Run them again in a new terminal when needed, or place them in a small setup script.

---

## 4. Step 2 — Obtain and Build the Linux Kernel

We cloned the upstream Linux kernel source into the `linux/` directory. The repository was cloned using a shallow clone, so it contains limited Git history.

From the project directory:

```bash
git clone --depth 1 https://github.com/torvalds/linux.git linux
```

We then entered the kernel directory:

```bash
cd ~/Desktop/embedded_linux/qemu/Orangepi3-qemu/linux
```

Set the build variables:

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-none-linux-gnu-
```

### Configure the kernel

```bash
make defconfig
```

This generates a default configuration suitable for the selected architecture.

The configuration is stored in:

```text
linux/.config
```

### Compile the kernel

```bash
make -j$(nproc)
```

- `make` builds the kernel and its required components.
- `-j$(nproc)` allows parallel compilation using the available CPU cores.

The resulting kernel image is:

```text
linux/arch/arm64/boot/Image
```

We confirmed that this image was approximately 50 MB.

### Important kernel configuration

Our kernel configuration includes:

```text
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
CONFIG_SERIAL_AMBA_PL011=y
CONFIG_SERIAL_AMBA_PL011_CONSOLE=y
```

These settings are relevant because:

- `CONFIG_DEVTMPFS=y` enables the device-node filesystem.
- `CONFIG_DEVTMPFS_MOUNT=y` allows the kernel to mount devtmpfs automatically in supported boot configurations.
- `CONFIG_SERIAL_AMBA_PL011=y` enables the PL011 UART driver.
- `CONFIG_SERIAL_AMBA_PL011_CONSOLE=y` allows the PL011 UART to serve as a kernel console.

In our initramfs setup, we explicitly mounted devtmpfs in the startup script rather than relying on automatic mounting.

---

## 5. Step 3 — Obtain and Configure BusyBox

BusyBox provides many basic Linux utilities in one executable, including:

- `sh`
- `ls`
- `mount`
- `mkdir`
- `cat`
- `echo`
- `uname`

We cloned BusyBox into the project:

```bash
cd ~/Desktop/embedded_linux/qemu/Orangepi3-qemu
git clone https://git.busybox.net/busybox busybox
```

Enter its source directory:

```bash
cd busybox
```

Set the cross-compilation environment:

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-none-linux-gnu-
```

### Generate the default configuration

```bash
make defconfig
```

### Enable static linking

```bash
make menuconfig
```

In the BusyBox configuration interface, enable:

```text
Settings
  → Build static binary (no shared libs)
```

The exact menu placement can vary slightly by BusyBox version.

Our configuration includes:

```text
CONFIG_STATIC=y
```

**Why static linking matters:**

A dynamically linked executable may require shared libraries and a dynamic linker inside the root filesystem. By building BusyBox statically, we avoid those dependencies for this executable.

This makes our minimal root filesystem easier to boot.

### Configure the cross-compiler prefix

In BusyBox configuration, set the cross-compiler prefix to:

```text
aarch64-none-linux-gnu-
```

### Compile BusyBox

```bash
make -j$(nproc)
```

### Install BusyBox into our root filesystem

```bash
make CONFIG_PREFIX=../rootfs install
```

This installs BusyBox and creates the appropriate command links inside `rootfs/`.

For example:

```text
rootfs/bin/busybox
rootfs/bin/sh -> busybox
rootfs/sbin/init -> ../bin/busybox
```

The `init` and `sh` entries are symbolic links to the same BusyBox executable. BusyBox selects the relevant functionality based on how it is invoked.

---

## 6. Step 4 — Create the Root Filesystem Directories

From the BusyBox directory, we created the additional directories required for initialization:

```bash
mkdir -p ../rootfs/{dev,proc,sys,etc,tmp}
```

Their purposes are:

- `/dev`: Device nodes, such as the UART terminal.
- `/proc`: A mount point for the proc filesystem.
- `/sys`: A mount point for the sysfs filesystem.
- `/etc`: System configuration files.
- `/tmp`: Temporary files.

These directories are mount points or locations for files; creating a directory does not itself mount a filesystem.

---

## 7. Step 5 — Configure the Startup Script (`rcS`)

We created the startup script at:

```text
rootfs/etc/init.d/rcS
```

Its contents are:

```sh
#!/bin/sh

mount -t devtmpfs devtmpfs /dev
mount -t proc proc /proc
mount -t sysfs sysfs /sys

echo "Embedded Linux started!"
```

### Explanation

**Line 1:**

```sh
#!/bin/sh
```

This tells the system to execute the script using the shell available at `/bin/sh`.

**Mount devtmpfs:**

```sh
mount -t devtmpfs devtmpfs /dev
```

This mounts devtmpfs at `/dev`, allowing the kernel-managed device nodes to be exposed there. In our QEMU environment, this allows `/dev/ttyAMA0` to appear.

**Mount proc:**

```sh
mount -t proc proc /proc
```

This exposes process and other kernel information through `/proc`.

**Mount sysfs:**

```sh
mount -t sysfs sysfs /sys
```

This exposes information about devices, drivers, and the kernel's device model through `/sys`.

**Print a startup message:**

```sh
echo "Embedded Linux started!"
```

This confirms that the startup script has run.

Make the script executable:

```bash
chmod +x ~/Desktop/embedded_linux/qemu/Orangepi3-qemu/rootfs/etc/init.d/rcS
```

---

## 8. Step 6 — Configure BusyBox Init (`inittab`)

We created the configuration file:

```text
rootfs/etc/inittab
```

Its contents are:

```text
::sysinit:/etc/init.d/rcS
ttyAMA0::askfirst:-/bin/sh
::restart:/sbin/init
```

BusyBox init reads this file to determine what actions to perform during initialization.

### First entry

```text
::sysinit:/etc/init.d/rcS
```

Runs the startup script during system initialization.

This is where we mount `/dev`, `/proc`, and `/sys`.

### Second entry

```text
ttyAMA0::askfirst:-/bin/sh
```

Starts an interactive shell on the `ttyAMA0` console after asking the user to press Enter.

This is why QEMU prints:

```text
Please press Enter to activate this console.
```

The `ttyAMA0` device is the virtual UART provided by QEMU's machine model and handled by the PL011 driver.

The `-` before `/bin/sh` tells the shell to behave as a login shell.

### Third entry

```text
::restart:/sbin/init
```

Tells BusyBox init what to do if the init process terminates. Here, it is configured to restart init.

---

## 9. Step 7 — Package the Root Filesystem as an Initramfs

Linux needs a filesystem containing the initial userspace programs and configuration files. We use an initramfs archive for this purpose.

We created the archive from inside the root filesystem directory:

```bash
cd ~/Desktop/embedded_linux/qemu/Orangepi3-qemu/rootfs
find . -print | cpio -H newc -o | gzip > ../rootfs.cpio.gz
```

### Explanation

**`find . -print`**

Lists the directory tree and its contents, starting from the current directory.

**`cpio -H newc -o`**

Packages the listed files into a `newc`-format CPIO archive.

**`gzip`**

Compresses the archive.

**`> ../rootfs.cpio.gz`**

Writes the compressed archive to the parent project directory.

The resulting file is:

```text
Orangepi3-qemu/rootfs.cpio.gz
```

### Why must we recreate the archive after modifying rootfs?

QEMU loads the archive, not our working `rootfs/` directory.

Therefore, if we change `rcS`, `inittab`, or another file, we must recreate `rootfs.cpio.gz` for the changes to be included in the next boot.

**Important:** Run the archive command from inside `rootfs/` so paths in the archive start with entries such as `./bin` and `./etc`, rather than an absolute path on our development PC.

---

## 10. Step 8 — Boot Linux with QEMU

First, go to the project directory:

```bash
cd ~/Desktop/embedded_linux/qemu/Orangepi3-qemu
```

Run:

```bash
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a53 \
  -m 512M \
  -kernel linux/arch/arm64/boot/Image \
  -initrd rootfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/sbin/init" \
  -nographic
```

### Explanation of each option

| Option | Purpose |
|---|---|
| `qemu-system-aarch64` | Starts the AArch64 system emulator. |
| `-machine virt` | Creates a generic virtual ARM machine. |
| `-cpu cortex-a53` | Selects a Cortex-A53 CPU model. |
| `-m 512M` | Assigns 512 MB of RAM to the virtual machine. |
| `-kernel .../Image` | Loads our compiled Linux kernel directly. |
| `-initrd rootfs.cpio.gz` | Loads the compressed initramfs. |
| `console=ttyAMA0` | Directs the kernel console to the virtual UART. |
| `rdinit=/sbin/init` | Tells the kernel to run BusyBox init from the initramfs. |
| `-nographic` | Uses the terminal for the virtual machine's serial console instead of opening a graphical display. |

### What happens during boot?

1. QEMU creates the virtual machine and loads the kernel and initramfs.
2. The kernel initializes memory management and supported virtual hardware.
3. The PL011 UART driver registers `ttyAMA0`.
4. The kernel unpacks the initramfs and makes its contents available as the initial root filesystem.
5. The kernel starts `/sbin/init`.
6. BusyBox init reads `/etc/inittab`.
7. BusyBox init runs `/etc/init.d/rcS`.
8. The startup script mounts devtmpfs, proc, and sysfs.
9. BusyBox init waits for Enter to start the interactive shell.

Unlike a typical physical-device boot, we are not configuring a bootloader in this exercise: QEMU loads the kernel and initramfs directly.

---

## 11. Step 9 — Verify the Running Linux System

After boot, our successful output was:

```text
Run /sbin/init as init process
Embedded Linux started!

Please press Enter to activate this console.
```

Press Enter to activate the shell.

### Verify the architecture

```sh
uname -m
```

Expected:

```text
aarch64
```

This confirms the running kernel reports the ARM64 architecture.

### Verify mounted filesystems

```sh
mount
```

Our successful output included:

```text
rootfs on / type rootfs (...)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
devtmpfs on /dev type devtmpfs (...)
```

This confirms that:

- The initial root filesystem is available at `/`.
- `/proc` is mounted.
- `/sys` is mounted.
- `/dev` is mounted using devtmpfs.

### Verify the UART device node

```sh
ls -l /dev/ttyAMA0
```

We expect a device entry beginning with `c`, indicating a character device.

The UART driver creates the device, and devtmpfs exposes the device node in `/dev`.

---

## 12. Troubleshooting We Encountered

### Problem: `can't open /dev/ttyAMA0: No such file or directory`

BusyBox init tried to start a shell on `ttyAMA0`, but the corresponding device node was missing.

We confirmed that the kernel supported devtmpfs and the PL011 UART driver.

We temporarily booted directly into a shell using:

```text
rdinit=/bin/sh
```

From that shell, we manually mounted the filesystems:

```sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
```

After mounting devtmpfs, `/dev/ttyAMA0` appeared.

We then added the devtmpfs mount to `rcS`, rebuilt the initramfs, and booted normally using `rdinit=/sbin/init`.

**Lesson learned:** Having a device driver enabled in the kernel does not guarantee that the corresponding device node will already be present in the root filesystem. The device filesystem must be available at the appropriate point during boot.

### Problem: QEMU option typo

We previously encountered a malformed command where `-m 512M` and `-kernel` were accidentally joined.

Make sure each option is separated by whitespace and use the multiline command documented above.

### Problem: Changes to `rcS` do not appear after reboot

Rebuild the initramfs after modifying the rootfs:

```bash
cd ~/Desktop/embedded_linux/qemu/Orangepi3-qemu/rootfs
find . -print | cpio -H newc -o | gzip > ../rootfs.cpio.gz
```

Then launch QEMU again from the project directory.

### Exit QEMU

When using `-nographic`, press:

1. `Ctrl+A`
2. Release both keys.
3. Press `X`.

---

## 13. Quick Reference — Reboot the System in the Future

Once the kernel, BusyBox, rootfs, and initramfs have been built, the usual procedure is simple.

### A. Open the project directory

```bash
cd ~/Desktop/embedded_linux/qemu/Orangepi3-qemu
```

### B. Launch QEMU

```bash
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a53 \
  -m 512M \
  -kernel linux/arch/arm64/boot/Image \
  -initrd rootfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/sbin/init" \
  -nographic
```

### C. Enter the shell

Press Enter when prompted.

### D. Check the environment

```sh
uname -m
mount
ls -l /dev/ttyAMA0
```

### E. Exit

Press `Ctrl+A`, then `X`.

**You do not need to rebuild the kernel or BusyBox every time you want to start QEMU.** Rebuild only when you change the corresponding source, configuration, or files included in the initramfs.

---

## 14. What We Have Learned

- How a cross-compiler targets ARM64 while running on an x86-64 PC.
- How to configure and compile a Linux kernel.
- The difference between a kernel image and a root filesystem.
- How BusyBox provides basic userspace utilities.
- How `/sbin/init` starts the initial userspace environment.
- How BusyBox init uses `/etc/inittab`.
- How an `rcS` script performs early userspace initialization.
- What `/dev`, `/proc`, and `/sys` are used for.
- Why the UART console requires both kernel support and an accessible device node.
- How to package a root filesystem into a compressed initramfs.
- How QEMU loads a kernel and initramfs directly.
- How to verify that the ARM64 Linux system has booted successfully.

## 15. Next Learning Steps

Our current milestone is complete: **we have a working ARM64 Linux kernel booting a minimal BusyBox userspace in QEMU, with an interactive serial console.**

The next logical steps are:

1. Explore Linux processes, PID 1, and shell commands.
2. Write a small C program on the host and cross-compile it for ARM64.
3. Copy the executable into `rootfs/`, rebuild the initramfs, and run it in QEMU.
4. Learn how Linux networking works and configure networking in QEMU.
5. Introduce Buildroot to generate a more complete embedded Linux system.
6. Learn about bootloaders, device trees, and kernel configuration for real hardware.
7. Transition from the generic QEMU machine to the Orange Pi 3 LTS.

The final target is to use these skills to develop the Linux software for our planned Ethernet intercom system.