This tutorial adapts Tomaž Zaman's professional, **`kas`**-based workflow to run entirely on **QEMU** for 64-bit ARM, using the **Scarthgap** release. This single-phase approach replaces manual environment scripts with a unified project specification and custom layers.

### 1. Project Foundation and Caching
Zaman emphasizes keeping build artifacts like the **Shared State Cache** and **Downloads** outside of the project directory to speed up future builds and keep the workspace clean. 

Run these commands to set up your external cache:
```bash
mkdir -p ~/yocto-cache/sstate-cache
mkdir -p ~/yocto-cache/downloads
mkdir -p ~/project-coding/yocto-pro
cd ~/project-coding/yocto-pro
```

### 2. The `kas` Orchestrator (`firmware.yml`)
Instead of manually cloning repositories as Chris Simmons does, you will use a **`kas`** YAML file. This tool, developed by Siemens, handles repository fetching and environment setup automatically.

Create a file named `firmware.yml` in your project root:
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
    # Bundling the rootfs into the kernel for a single-file boot
    INITRAMFS_IMAGE = "core-image-base"
    INITRAMFS_IMAGE_BUNDLE = "1"
    IMAGE_INSTALL:append = " nano"
    # Throttle for WSL2 stability [Manual Add]
    BB_NUMBER_THREADS = "4"
    PARALLEL_MAKE = "-j 4"
```

### 3. Creating the Custom Layer
Professional Yocto development happens in custom layers rather than editing base files. Create the directory structure for your layer:

```bash
mkdir -p meta-custom/conf
mkdir -p meta-custom/recipes-kernel/linux/files
```

**File: `meta-custom/conf/layer.conf`**
Paste this standard configuration to tell Bitbake where to find your recipes:
```text
BBPATH .= ":${LAYERDIR}"
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-custom"
BBFILE_PATTERN_meta-custom = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-custom = "6"

LAYERSERIES_COMPAT_meta-custom = "scarthgap"
```

### 4. Customizing the Kernel (The .bbappend)
Zaman demonstrates that you can modify the kernel using a **`.bbappend`** file. This allows you to inject custom configurations or patches without rewriting the entire kernel recipe.

**File: `meta-custom/recipes-kernel/linux/linux-yocto_%.bbappend`**
```text
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

# Example: Prepending a task to verify setup
do_configure:prepend() {
    echo "Configuring custom QEMU kernel..."
}
```

### 5. Global Cache Configuration (`site.conf`)
To ensure Bitbake uses the external cache directories you created in Step 1, create a `site.conf`. This file is automatically loaded by Bitbake if present.

**File: `site.conf`**
(Place this inside the directory that `kas` will create, typically `build/conf/site.conf`, or define it in your `kas` file under `local_conf_header` as shown in Step 2).
```text
SSTATE_DIR = "/home/youruser/yocto-cache/sstate-cache"
DL_DIR = "/home/youruser/yocto-cache/downloads"
```

### 6. Build and Run
Zaman’s workflow uses `kas build` to handle the entire process from fetching code to final compilation.

```bash
# Ensure kas is installed (pip install kas)
kas build firmware.yml
```

Once the build finishes, your image is "bundled"—the root filesystem is embedded directly into the kernel. Launch it using the standard QEMU wrapper:

```bash
# Enter the kas shell to access Yocto tools
kas shell firmware.yml -c "runqemu qemuarm64 nographic"
```

*   **Login:** `root` (no password).
*   **Verify:** Run `nano` to confirm the package was appended correctly.
*   **Exit:** Type `poweroff` or use `Ctrl+a` then `x` to kill the emulator [Source 1].

Would you like me to find more sources on how to optimize your `kas` configuration for multi-developer teams?