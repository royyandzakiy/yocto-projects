## Phase 1: Build and Run the PREEMPT_RT Image

This phase sets up your environment variables, configures the build system to swap the standard kernel for a real-time patched kernel (`PREEMPT_RT`), and boots it into QEMU.

### 1. Initialize the Build Environment

Run this from your main Yocto project directory to jump into your dedicated build folder.

```bash
cd ~/project-coding/yocto
source poky/oe-init-build-env builds/qemuarm64

```

### 2. Configure target, RT kernel, and resources in local.conf

Open `conf/local.conf` in a text editor.

* Find or modify the `MACHINE` variable to target ARM64:

```bitbake
MACHINE ??= "qemuarm64"

```

* **[ADDED STEP FOR RT]** Switch the default kernel provider and kernel selection from standard to real-time. Append these configurations:

```bitbake
# Use the real-time kernel provider
PREFERRED_PROVIDER_virtual/kernel = "linux-yocto-rt"
PREFERRED_VERSION_linux-yocto-rt = "6.6%"

# Include debugging and real-time validation tools (optional but highly recommended)
IMAGE_INSTALL:append = " rt-tests"

```

* Scroll to the absolute bottom of the file and add these lines to throttle resource consumption to prevent the host from running out of RAM:

```bitbake
BB_NUMBER_THREADS = "4"
PARALLEL_MAKE = "-j 4"

```

### 3. Build the PREEMPT_RT Image

Kick off the compilation for your base image. Because you changed the kernel provider, Yocto will automatically download, patch, and build the `linux-yocto-rt` kernel instead of `linux-yocto-standard`.

```bash
bitbake core-image-base

```

#### 3.1. If along the way fail, might need to start from cleanstate

*(Note: Because we changed the kernel package name, if you hit a cross-compiler cache collision, clean the toolchain)*

```bash
bitbake -c cleansstate gcc-cross-aarch64

```

### 4. Boot the Real-Time System

Once the build completes successfully, launch your 64-bit real-time ARM Linux system directly in your terminal:

```bash
runqemu qemuarm64 nographic

```

* **Login:** Type `root` (there is no password).

### 5. Verify PREEMPT_RT Status

Once inside the emulated Linux prompt, run the following commands to ensure your kernel is truly real-time:

#### 5.1. Check Kernel Version and Build Type

Run `uname -a`. You should explicitly look for **`PREEMPT_RT`** or **`rt`** in the kernel string.

```bash
root@qemuarm64:~# uname -a

```

*Expected output snippet:*

> Linux qemuarm64 6.6.x-yocto-**rt** ... SMP **PREEMPT_RT** ...

#### 5.2. Run a Quick Latency Test

Since we appended `rt-tests` to `IMAGE_INSTALL`, you can use `cyclictest` to test the real-time deterministic behavior of your threads:

```bash
root@qemuarm64:~# cyclictest -l1000 -m -n -p99

```

### 6. Exit the Emulator

Type `poweroff` inside the emulated prompt to safely close QEMU.

```bash
root@qemuarm64:~# poweroff

```