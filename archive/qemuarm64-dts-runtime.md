Excellent idea! Runtime DTS manipulation is powerful for learning. Here's how to add/modify device tree while the system is running using Device Tree Overlays.



Runtime DTS Modification Steps



Step 1: Enable Overlay Support in Kernel (Before Build)



Add to your conf/local.conf:



```bash

cat >> conf/local.conf << 'EOF'

# Enable runtime overlay support

KERNEL_CONFIG_FRAGMENTS += " ${TOPDIR}/../meta-custom/recipes-kernel/linux/files/overlays.cfg"



# Install dtc and overlay utilities

CORE_IMAGE_EXTRA_INSTALL += "device-tree-compiler"

EOF

```



Create the kernel config fragment:



```bash

cat > meta-custom/recipes-kernel/linux/files/overlays.cfg << 'EOF'

CONFIG_OF_OVERLAY=y

CONFIG_OF_CONFIGFS=y

CONFIG_OF_DYNAMIC=y

CONFIG_DEBUG_FS=y

CONFIG_OVERLAY_FS=y

EOF

```



Step 2: Rebuild with Overlay Support



```bash

bitbake -c cleansstate virtual/kernel

bitbake core-image-base

```



Step 3: RUNTIME OPERATIONS - Inside Running QEMU



Boot your system normally:



```bash

runqemu qemuarm64 nographic

```



Now you can manipulate DTS at runtime:



A. Check current device tree



```bash

root@qemuarm64:~# ls /sys/firmware/devicetree/base/

root@qemuarm64:~# cat /sys/firmware/devicetree/base/compatible

# Output: linux,dummy-virt

```



B. Create a runtime LED overlay



```bash

root@qemuarm64:~# cat > /tmp/runtime-led.dts << 'EOF'

/dts-v1/;

/plugin/;



/ {

    fragment@0 {

        target-path = "/";

        __overlay__ {

            runtime-leds {

                compatible = "gpio-leds";

                runtime-led {

                    label = "runtime-led";

                    gpios = <&gpio 13 0>;

                    default-state = "on";

                };

            };

        };

    };

};

EOF

```



C. Compile and apply the overlay live



```bash

# Compile DTS to overlay blob

root@qemuarm64:~# dtc -@ -I dts -O dtb /tmp/runtime-led.dts -o /tmp/runtime-led.dtbo



# Check configfs is mounted

root@qemuarm64:~# mount | grep configfs

# If not mounted: mount -t configfs configfs /sys/kernel/config



# Apply overlay

root@qemuarm64:~# mkdir -p /sys/kernel/config/device-tree/overlays/my-led

root@qemuarm64:~# cat /tmp/runtime-led.dtbo > /sys/kernel/config/device-tree/overlays/my-led/dtbo



# Verify it loaded

root@qemuarm64:~# ls /sys/class/leds/

# You should now see 'runtime-led' alongside any others

```



D. Control the runtime-added LED



```bash

# Turn on runtime LED

root@qemuarm64:~# echo 1 > /sys/class/leds/runtime-led/brightness



# Make it blink

root@qemuarm64:~# echo heartbeat > /sys/class/leds/runtime-led/trigger

```



E. Remove overlay (unload DTS changes)



```bash

# Remove overlay (LED disappears)

root@qemuarm64:~# rmdir /sys/kernel/config/device-tree/overlays/my-led



# Verify LED is gone

root@qemuarm64:~# ls /sys/class/leds/  # runtime-led should be gone

```



Step 4: Advanced Runtime Example - Add I2C Device



```bash

root@qemuarm64:~# cat > /tmp/i2c-device.dts << 'EOF'

/dts-v1/;

/plugin/;



&i2c0 {

    #address-cells = <1>;

    #size-cells = <0>;

    

    temp-sensor@48 {

        compatible = "lm75";

        reg = <0x48>;

        status = "okay";

    };

};

EOF



# Compile and apply

root@qemuarm64:~# dtc -@ -I dts -O dtb /tmp/i2c-device.dts -o /tmp/i2c-device.dtbo

root@qemuarm64:~# mkdir -p /sys/kernel/config/device-tree/overlays/i2c

root@qemuarm64:~# cat /tmp/i2c-device.dtbo > /sys/kernel/config/device-tree/overlays/i2c/dtbo



# Check if device appeared

root@qemuarm64:~# ls /sys/bus/i2c/devices/

```



Step 5: Runtime Debug Techniques



```bash

# View currently applied overlays

root@qemuarm64:~# ls /sys/kernel/config/device-tree/overlays/



# Check device tree status after overlay

root@qemuarm64:~# find /proc/device-tree/ -name "*runtime-led*"



# Monitor kernel messages during overlay application

root@qemuarm64:~# dmesg | tail -20



# Check for overlay errors

root@qemuarm64:~# dtc -@ -I dts -O dtb /tmp/runtime-led.dts 2>&1

```



Step 6: Create Reusable Runtime Script



Save this as /usr/bin/runtime-overlay on your target:



```bash

root@qemuarm64:~# cat > /usr/bin/runtime-overlay << 'EOF'

#!/bin/sh

case "$1" in

    apply)

        dtc -@ -I dts -O dtb $2 -o /tmp/overlay.dtbo

        mkdir -p /sys/kernel/config/device-tree/overlays/$3

        cat /tmp/overlay.dtbo > /sys/kernel/config/device-tree/overlays/$3/dtbo

        echo "Overlay $3 applied"

        ;;

    remove)

        rmdir /sys/kernel/config/device-tree/overlays/$2

        echo "Overlay $2 removed"

        ;;

    list)

        ls /sys/kernel/config/device-tree/overlays/

        ;;

    *)

        echo "Usage: runtime-overlay {apply <dts-file> <name>|remove <name>|list}"

        ;;

esac

EOF



root@qemuarm64:~# chmod +x /usr/bin/runtime-overlay

```



Use it:



```bash

root@qemuarm64:~# runtime-overlay apply /tmp/runtime-led.dts myled

root@qemuarm64:~# runtime-overlay list

root@qemuarm64:~# runtime-overlay remove myled

```



Important Notes for Runtime DTS



1. Not all devices support runtime addition - GPIO/LEDs work well, interrupt controllers don't

2. Symbols required - Base DTB must be compiled with -@ flag (enabled by our kernel config)

3. Memory can't be freed - Removing overlays leaves some memory allocated

4. QEMU limitation - Some hardware changes require reset, but GPIO/LEDs work perfectly



Testing Without Rebuild



The beauty of runtime DTS:



```bash

# Edit DTS live, see changes immediately

root@qemuarm64:~# vi /tmp/test.dts

root@qemuarm64:~# runtime-overlay apply /tmp/test.dts test1

# See if it works...

root@qemuarm64:~# runtime-overlay remove test1

# Edit again, reapply - no rebuild needed!

```



This gives you instant feedback for learning DTS syntax and structure without 2-hour Yocto rebuilds. Perfect for experimentation!