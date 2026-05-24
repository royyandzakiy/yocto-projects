Perfect! I'll use that link as your baseline. Here's how to add a custom LED device tree node before building the image, modifying the steps from your Phase 1 guide.



Modified Workflow: Add LED to DTS Before Building



Step 1: Initial Setup (same as your guide)



```bash

cd ~/project-coding/yocto

source poky/oe-init-build-env builds/qemuarm64

```



Step 2: Configure local.conf (same as your guide)



```bash

# Edit conf/local.conf - set these

MACHINE ??= "qemuarm64"

BB_NUMBER_THREADS = "4"

PARALLEL_MAKE = "-j 4"

```



Step 3: NEW - Create Device Tree Override Layer



Instead of modifying kernel directly, create a custom layer:



```bash

# Create layer structure

mkdir -p meta-custom/recipes-kernel/linux/files

mkdir -p meta-custom/recipes-kernel/linux/linux-yocto



# Create the device tree overlay

cat > meta-custom/recipes-kernel/linux/files/my-led-overlay.dts << 'EOF'

/dts-v1/;

/plugin/;



/ {

    fragment@0 {

        target = <&gpio>;

        __overlay__ {

            led-controller {

                compatible = "gpio-leds";

                led-0 {

                    label = "user-led";

                    gpios = <&gpio 12 0>;  // GPIO12, active high

                    default-state = "off";

                    linux,default-trigger = "heartbeat";

                };

            };

        };

    };

};

EOF

```



Step 4: NEW - Create Kernel Recipe Append



```bash

# Create bbappend to add DTS

cat > meta-custom/recipes-kernel/linux/linux-yocto_6.6.bbappend << 'EOF'

FILESEXTRAPATHS:prepend := "${THISDIR}/files:"



SRC_URI:append = " file://my-led-overlay.dts"



do_configure:append() {

    # Compile overlay with kernel build

    dtc -@ -I dts -O dtb -o ${B}/arch/arm64/boot/dts/my-led-overlay.dtbo \

        ${WORKDIR}/my-led-overlay.dts

}



do_install:append() {

    install -d ${D}/boot/overlays

    install -m 0644 ${B}/arch/arm64/boot/dts/my-led-overlay.dtbo ${D}/boot/overlays/

}

EOF

```



Step 5: NEW - Create LED Test Application Recipe



```bash

# Create LED control app

mkdir -p meta-custom/recipes-app/led-test/files



cat > meta-custom/recipes-app/led-test/files/led-test.c << 'EOF'

#include <stdio.h>

#include <stdlib.h>

#include <fcntl.h>

#include <unistd.h>

#include <string.h>



int main() {

    FILE *fp;

    char buf[64];

    

    // Trigger heartbeat via sysfs

    fp = fopen("/sys/class/leds/user-led/trigger", "w");

    if (fp) {

        fprintf(fp, "heartbeat");

        fclose(fp);

        printf("LED set to heartbeat mode!\n");

        printf("Watch the LED blink...\n");

        sleep(10);

        

        // Change to timer mode

        fp = fopen("/sys/class/leds/user-led/trigger", "w");

        fprintf(fp, "timer");

        fclose(fp);

        printf("Now in timer mode (500ms on/off)\n");

        sleep(5);

        

        // Turn off

        fp = fopen("/sys/class/leds/user-led/brightness", "w");

        fprintf(fp, "0");

        fclose(fp);

        printf("LED off\n");

    } else {

        printf("LED not found - kernel missing gpio-leds driver?\n");

        return 1;

    }

    return 0;

}

EOF



# Create recipe

cat > meta-custom/recipes-app/led-test/led-test.bb << 'EOF'

SUMMARY = "LED test application"

LICENSE = "MIT"

LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"



SRC_URI = "file://led-test.c"



S = "${WORKDIR}"



do_compile() {

    ${CC} ${CFLAGS} ${LDFLAGS} led-test.c -o led-test

}



do_install() {

    install -d ${D}${bindir}

    install -m 0755 led-test ${D}${bindir}/

}

EOF

```



Step 6: NEW - Add Layer to Build



```bash

# Add meta-custom to bblayers.conf

bitbake-layers add-layer ../meta-custom



# Enable LED overlay in kernel config (optional but good)

cat >> conf/local.conf << 'EOF'

KERNEL_DEVICETREE:append = " my-led-overlay.dtbo"



# Ensure GPIO LED driver is built-in

KERNEL_CONFIG_FRAGMENTS += " ${TOPDIR}/../meta-custom/recipes-kernel/linux/files/leds.cfg"

EOF



# Create kernel config fragment

mkdir -p meta-custom/recipes-kernel/linux/files

cat > meta-custom/recipes-kernel/linux/files/leds.cfg << 'EOF'

CONFIG_LEDS_GPIO=y

CONFIG_LEDS_CLASS=y

CONFIG_NEW_LEDS=y

EOF

```



Step 7: Build Modified Image (continued from your Step 3)



```bash

# Clean kernel to force rebuild with overlay

bitbake -c cleansstate virtual/kernel



# Build everything (same as your step 3, but now with LED)

bitbake core-image-base

```



Step 8: Boot and Test (modified from your Step 4)



```bash

# Run with overlay applied

runqemu qemuarm64 nographic



# Inside QEMU as root:

root@qemuarm64:~# ls /sys/class/leds/

# You should see 'user-led'



# Test with our app

root@qemuarm64:~# led-test

# LED should blink in heartbeat pattern



# Or manually control:

root@qemuarm64:~# echo 1 > /sys/class/leds/user-led/brightness  # LED on

root@qemuarm64:~# echo 0 > /sys/class/leds/user-led/brightness  # LED off

root@qemuarm64:~# echo heartbeat > /sys/class/leds/user-led/trigger

```



Why This Works for Learning DTS



1. No hardware needed - QEMU virt board emulates GPIO controllers

2. Overlays are safe - Changes don't break base kernel

3. See immediate results - LED appears in /sys/class/leds/

4. Matches real workflow - This is exactly how production Yocto adds DTS changes



Debug Tips if LED Doesn't Appear



```bash

# Inside QEMU, check if overlay loaded

root@qemuarm64:~# find /proc/device-tree/ -name "*led*"

root@qemuarm64:~# dmesg | grep -i gpio



# On host, verify DTB compilation

dtc -I dtb -O dts tmp/deploy/images/qemuarm64/qemu-arm64.dtb | grep -A10 leds

```



This gives you a complete, working DTS modification flow using your exact Yocto setup. The LED appears as a virtual device in QEMU - perfect for learning without real hardware!