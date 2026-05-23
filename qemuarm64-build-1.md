Phase 1 focuses entirely on getting your baseline `core-image-base` compiled and booting in QEMU for the first time. Phase 2 walks you through modifying that baseline by adding the open-source layers, your custom metadata layer, and recompiling.

---

## Phase 1: Build and Run the Baseline Image

This phase sets up your environment variables, optimizes your system constraints, and builds a standard ARM64 Linux image from source.

### 1. Initialize the Build Environment

Run this from your main Yocto project directory to jump into your dedicated build folder.

```bash
cd ~/project-coding/yocto
source poky/oe-init-build-env builds/qemuarm64
```

### 2. Configure target and resources in local.conf

Open `conf/local.conf` in a text editor (like VS Code or nano).

* Find the `MACHINE` variable and modify it to target ARM64:

```bitbake
MACHINE ??= "qemuarm64"
```

* Scroll to the absolute bottom of the file and add these lines to throttle resource consumption. This prevents the WSL2 Linux kernel from crashing due to running out of RAM:

```bitbake
  BB_NUMBER_THREADS = "4"
  PARALLEL_MAKE = "-j 4"
```

### 3. Build the Baseline Image

The speaker recommends **core-image-base** over `minimal` because it supports features like kernel modules. Kick off the compilation:

```bash
bitbake core-image-base
```

#### 3.1. If along the way fail, might need to start from cleanstate

```bash
bitbake -c cleansstate gcc-cross-aarch64
```

*(This first compilation builds the entire cross-toolchain and Linux distribution from scratch. It will take a few hours).*

### 4. Boot the Baseline System

Once the build completes successfully, launch your 64-bit ARM Linux system directly in your terminal:

```bash
runqemu qemuarm64 nographic

```

* **Login:** Type `root` (there is no password).
* **Verify:** Try typing `nano` or `hello`—you will see they do not exist yet.
* **Exit:** Type `poweroff` inside the emulated prompt to close QEMU safely.

### 5. Success

Successfully creating image
```bash

...

Loading cache: 100% |########################################################################################################################################################| Time: 0:00:00
Loaded 1878 entries from dependency cache.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "2.8.1"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "aarch64-poky-linux"
MACHINE              = "qemuarm64"
DISTRO               = "poky"
DISTRO_VERSION       = "5.0.17"
TUNE_FEATURES        = "aarch64 crc cortexa57"
TARGET_FPU           = ""
meta
meta-poky
meta-yocto-bsp       = "scarthgap:cb2dcb4963e5fbe449f1bcb019eae883ddecc8ec"

Sstate summary: Wanted 710 Local 0 Mirrors 0 Missed 710 Current 1139 (0% match, 61% complete)################################################################                | ETA:  0:00:00
Initialising tasks: 100% |###################################################################################################################################################| Time: 0:00:02
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 4060 tasks of which 2545 didn't need to be rerun and all succeeded.
```

Running image on qemuarm64
```bash
royya@tuff16:~/project-coding/yocto/builds/qemuarm64$ runqemu qemuarm64 nographic
runqemu - INFO - Running MACHINE=qemuarm64 bitbake -e  ...
runqemu - INFO - Continuing with the following parameters:
KERNEL: [/home/royya/project-coding/yocto/builds/qemuarm64/tmp/deploy/images/qemuarm64/Image]
MACHINE: [qemuarm64]
FSTYPE: [ext4]
ROOTFS: [/home/royya/project-coding/yocto/builds/qemuarm64/tmp/deploy/images/qemuarm64/core-image-minimal-qemuarm64.rootfs-20260523043951.ext4]
CONFFILE: [/home/royya/project-coding/yocto/builds/qemuarm64/tmp/deploy/images/qemuarm64/core-image-minimal-qemuarm64.rootfs-20260523043951.qemuboot.conf]

runqemu - INFO - Setting up tap interface under sudo
[sudo] password for royya:
runqemu - INFO - Network configuration: ip=192.168.7.2::192.168.7.1:255.255.255.0::eth0:off:8.8.8.8 net.ifnames=0
runqemu - INFO - Running /home/royya/project-coding/yocto/builds/qemuarm64/tmp/work/x86_64-linux/qemu-helper-native/1.0/recipe-sysroot-native/usr/bin/qemu-system-aarch64 -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:02 -netdev tap,id=net0,ifname=tap0,script=no,downscript=no -object rng-random,filename=/dev/urandom,id=rng0 -device virtio-rng-pci,rng=rng0 -drive id=disk0,file=/home/royya/project-coding/yocto/builds/qemuarm64/tmp/deploy/images/qemuarm64/core-image-minimal-qemuarm64.rootfs-20260523043951.ext4,if=none,format=raw -device virtio-blk-pci,drive=disk0 -device qemu-xhci -device usb-tablet -device usb-kbd  -machine virt -cpu cortex-a57 -smp 4 -m 256 -serial mon:stdio -serial null -nographic -device virtio-gpu-pci -kernel /home/royya/project-coding/yocto/builds/qemuarm64/tmp/deploy/images/qemuarm64/Image -append 'root=/dev/vda rw  mem=256M ip=192.168.7.2::192.168.7.1:255.255.255.0::eth0:off:8.8.8.8 net.ifnames=0 console=ttyAMA0 console=hvc0 swiotlb=0 '

runqemu - INFO - Host uptime: 37730.52

[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x411fd070]
[    0.000000] Linux version 6.6.123-yocto-standard (oe-user@oe-host) (aarch64-poky-linux-gcc (GCC) 13.4.0, GNU ld (GNU Binutils) 2.42.0.20240723) #1 SMP PREEMPT Mon Feb  9 22:50:49 UTC 2026
[    0.000000] random: crng init done
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] Memory limited to 256MB
[    0.000000] efi: UEFI not found.
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000040000000-0x000000004fffffff]
[    0.000000]   DMA32    empty

...

[    1.567003] devtmpfs: mounted
[    1.582392] usb 1-1: new high-speed USB device number 2 using xhci_hcd
[    1.626958] Freeing unused kernel memory: 4608K
[    1.629461] Run /sbin/init as init process
[    1.743291] input: QEMU QEMU USB Tablet as /devices/platform/4010000000.pcie/pci0000:00/0000:00:04.0/usb1/1-1/1-1:1.0/0003:0627:0001.0001/input/input0
[    1.747620] hid-generic 0003:0627:0001.0001: input: USB HID v0.01 Mouse [QEMU QEMU USB Tablet] on usb-0000:00:04.0-1/input0
INIT: version 3.04 booting
[    1.874438] usb 1-2: new high-speed USB device number 3 using xhci_hcd
[    2.029911] input: QEMU QEMU USB Keyboard as /devices/platform/4010000000.pcie/pci0000:00/0000:00:04.0/usb1/1-2/1-2:1.0/0003:0627:0001.0002/input/input1
[    2.102705] hid-generic 0003:0627:0001.0002: input: USB HID v1.11 Keyboard [QEMU QEMU USB Keyboard] on usb-0000:00:04.0-2/input0
Starting udev
[    3.746250] udevd[112]: starting version 3.2.14
[    3.849055] udevd[113]: starting eudev-3.2.14
[    4.871638] EXT4-fs (vda): re-mounted 15c45318-7992-41bd-9aaa-e3887d5b7549.
INIT: Entering runlevel: 5
Configuring network interfaces... ip: RTNETLINK answers: File exists
Starting syslogd/klogd: done

Poky (Yocto Project Reference Distro) 5.0.17 qemuarm64 /dev/ttyAMA0

# here insert root to login, then will run inside linux as root like normal

qemuarm64 login: root

WARNING: Poky is a reference Yocto Project distribution that should be used for
testing and development purposes only. It is recommended that you create your
own distribution for production use.

root@qemuarm64:~# ls
root@qemuarm64:~# ls ..
root
root@qemuarm64:~# ls
root@qemuarm64:~# ls -a
.             ..            .ash_history
root@qemuarm64:~#

...

# run poweroff to end session

root@qemuarm64:~# poweroff

Broadcast message from root@qemuarm64 (ttyAMA0) (Sat May 23 06:11:20 2026):

The system is going down for system halt NOW!
INIT: Switching to runlevel: 0
INIT: Sending processes configured via /etc/inittab the TERM signal
Stopping syslogd/klogd: stopped syslogd (pid 265)
stopped klogd (pid 268)
done
Unmounting remote filesystems...
Deconfiguring network interfaces... ifdown: interface lo not configured
done.
Sending all processes the TERM signal...
Sending all processes the KILL signal...
Deactivating swap...
Unmounting local filesystems...
[  485.160610] EXT4-fs (vda): re-mounted 15c45318-7992-41bd-9aaa-e3887d5b7549 ro.
[  485.374322] reboot: Power down
runqemu - INFO - Cleaning up
runqemu - INFO - Host uptime: 38222.72

```