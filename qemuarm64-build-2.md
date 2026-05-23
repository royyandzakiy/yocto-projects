## Phase 2: Add Custom Layers and Applications

Now that your baseline system is verified, you can pull in external packages (`nano`) and write your own custom software ("Hello World").

### 1. Re-verify Environment

If you closed your terminal window, make sure you are re-initialized inside your build directory:

```bash
cd ~/project-coding/yocto
source poky/oe-init-build-env builds/qemuarm64
```

### 2. Add the Meta-OE Layer for Nano

The speaker notes that `nano` is located in the **meta-openembedded** repository. Add its sub-layer to your build configuration:

```bash
bitbake-layers add-layer ../../meta-openembedded/meta-oe
```

### 3. Create and Add your Custom Layer

Follow the speaker's "Hello World" demonstration by creating your own metadata layer to store your personal application recipes.

```bash
bitbake-layers create-layer ../../meta-example
bitbake-layers add-layer ../../meta-example
```

### 4. Write the Hello World Recipe

Create a new file at `../../meta-example/recipes-example/hello/hello_1.0.bb` and insert this code:

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

Next, create a nested subfolder for your source file:

```bash
mkdir -p ../../meta-example/recipes-example/hello/files

```

Create `../../meta-example/recipes-example/hello/files/hello.c` and add a basic main block:

```c
#include <stdio.h>

int main(void) {
    printf("Hello from your custom Yocto Layer!\n");
    return 0;
}
```

### 5. Inject the Packages into your Image Configuration

Open your `conf/local.conf` file again. Add these lines at the bottom to explicitly tell the build system to install `nano` and your custom `hello` executable into the final root filesystem:

```bitbake
IMAGE_INSTALL:append = " nano hello"
```

### 6. Recompile and Run

Run the build tool again. Because Yocto uses an incremental cache engine, **this run will only take a few minutes** because it only compiles your new layers and packs them into the image.

```bash
bitbake core-image-base
```

Once done, spin up your updated emulator:

```bash
runqemu qemuarm64 nographic
```

You can now run `nano` to edit files or type `hello` to see your custom C program execute!

```bash
root@qemuarm64:~# hello
Hello from your custom Yocto Layer!
```

### 7. Exit

Exit with `poweroff` or `Ctrl-a` then `x`.