I'll help you create a modified version that's safe and practical for learning on WSL2. The key changes are removing the kernel bundling and ensuring WSL2 compatibility.

## Modified Tutorial: Standard QEMU Build for WSL2

### 1. Project Setup with External Caching

```bash
mkdir -p ~/yocto-cache/sstate-cache
mkdir -p ~/yocto-cache/downloads
mkdir -p ~/project-coding/yocto-pro
cd ~/project-coding/yocto-pro
```

### 2. The `kas` Configuration (`firmware.yml`)

Create `firmware.yml` in your project root. This is the standard approach - no kernel bundling:

```yaml
header:
  version: 14

machine: qemuarm64
distro: poky
target: core-image-base

repos:
  poky:
    url: "git://git.yoctoproject.org/poky"
    branch: scarthgap
    layers:
      meta:
      meta-poky:
      meta-yocto-bsp:

  meta-openembedded:
    url: "git://git.openembedded.org/meta-openembedded"
    branch: scarthgap
    layers:
      meta-oe:

  meta-custom:
    path: meta-custom

local_conf_header:
  standard: |
    CONF_VERSION = "2"
    PACKAGE_CLASSES = "package_rpm"
    SDKMACHINE = "x86_64"
    IMAGE_INSTALL:append = " nano"
    
  wsl2_tuning: |
    # Critical for WSL2 stability
    BB_NUMBER_THREADS = "4"
    PARALLEL_MAKE = "-j 4"
    
    # External cache paths
    SSTATE_DIR = "${TOPDIR}/../../yocto-cache/sstate-cache"
    DL_DIR = "${TOPDIR}/../../yocto-cache/downloads"
```

### 3. Custom Layer Setup

```bash
mkdir -p meta-custom/conf
mkdir -p meta-custom/recipes-example/hello/files
```

**File: `meta-custom/conf/layer.conf`**
```text
BBPATH .= ":${LAYERDIR}"
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-custom"
BBFILE_PATTERN_meta-custom = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-custom = "6"

LAYERSERIES_COMPAT_meta-custom = "scarthgap"
```

### 4. Hello World Recipe

**File: `meta-custom/recipes-example/hello/hello_1.0.bb`**
```text
SUMMARY = "Simple hello world application"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://hello.c"

S = "${WORKDIR}"

do_compile() {
    ${CC} hello.c ${LDFLAGS} -o hello
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 hello ${D}${bindir}
}
```

**File: `meta-custom/recipes-example/hello/files/hello.c`**
```c
#include <stdio.h>

int main(void) {
    printf("Hello from your custom Yocto Layer!\n");
    return 0;
}
```

### 5. Add Hello to the Image

Update `firmware.yml` to include hello in `IMAGE_INSTALL`:

```yaml
local_conf_header:
  standard: |
    CONF_VERSION = "2"
    PACKAGE_CLASSES = "package_rpm"
    SDKMACHINE = "x86_64"
    IMAGE_INSTALL:append = " nano hello"
    
  # ... rest stays the same
```

### 6. Build the Image

First, install `kas` if you haven't:

```bash
sudo apt install kas # or pip install kas
```

Then build (this will take 1-3 hours on first run):

```bash
kas build firmware.yml
```

### 7. Run in QEMU

After the build completes:

```bash
kas shell firmware.yml -c "runqemu qemuarm64 nographic"
```

Login as `root` (no password) and test:
```bash
hello
nano test.txt
```

Exit with `poweroff` or `Ctrl-a` then `x`.

## Key Changes Explained

**Removed kernel bundling:** The original `INITRAMFS_IMAGE` lines are gone. Your image now produces a separate kernel and rootfs, which is standard and easier to debug.

**Relative cache paths:** Using `${TOPDIR}` avoids hardcoding your username, making the setup more portable.

**WSL2 tuning preserved:** The thread limits prevent memory issues on WSL2.

This approach gives you the professional `kas` workflow while keeping things simple enough to learn the tools properly. The first build will download everything and cache it externally, so subsequent builds or projects will be much faster.

Would you like me to explain any part of this setup in more detail?