I'll modify the Yocto setup to create a custom bare-metal style image called my-custom-image with:

1. Hello World C++ app
2. DHT22 sensor over I2C with device tree overlay
3. All existing SSH/Ethernet features preserved

---

Custom Yocto Image for Raspberry Pi 4B+ with I2C DHT22 Sensor

Directory Structure (All in poky/)

```bash
cd ~/yocto-rpi4/poky

# Your existing layers:
# - meta/
# - meta-raspberrypi/
# - meta-openembedded/

# Create custom layer for your app and device tree
bitbake-layers create-layer meta-custom
bitbake-layers add-layer meta-custom

cd meta-custom
```

1. Create Directory Structure

```bash
cd ~/yocto-rpi4/poky/meta-custom

mkdir -p recipes-kernel/device-tree/files
mkdir -p recipes-apps/helloworld/files
mkdir -p recipes-core/images
mkdir -p conf/machine
```

2. Create Custom Image Recipe

File: meta-custom/recipes-core/images/my-custom-image.bb

```bitbake
SUMMARY = "Custom Raspberry Pi 4B+ Image with I2C DHT22 and HelloWorld App"
DESCRIPTION = "Bare-metal style image with SSH, Ethernet, I2C DHT22 sensor support"
LICENSE = "MIT"

inherit core-image

# Include base features from your config
IMAGE_FEATURES += "ssh-server-dropbear allow-empty-password empty-root-password allow-root-login"

# Add packages
IMAGE_INSTALL += " \
    helloworld \
    i2c-tools \
    python3-core \
    python3-smbus \
    python3-pip \
    kernel-module-i2c-bcm2835 \
    kernel-module-i2c-dev \
    kernel-devicetree \
    libgpiod \
    gpiod \
"

# Enable I2C in config.txt on boot
ROOTFS_POSTPROCESS_COMMAND += "enable_i2c_and_dt;"

enable_i2c_and_dt() {
    # Enable I2C interface
    echo "dtparam=i2c_arm=on" >> ${IMAGE_ROOTFS}/boot/config.txt
    echo "dtparam=i2c1=on" >> ${IMAGE_ROOTFS}/boot/config.txt
    
    # Load I2C kernel modules
    echo "i2c-dev" >> ${IMAGE_ROOTFS}/etc/modules-load.d/i2c.conf
    echo "i2c-bcm2835" >> ${IMAGE_ROOTFS}/etc/modules-load.d/i2c.conf
    
    # Create I2C device node
    mkdir -p ${IMAGE_ROOTFS}/etc/udev/rules.d/
    echo 'SUBSYSTEM=="i2c-dev", MODE="0666"' > ${IMAGE_ROOTFS}/etc/udev/rules.d/99-i2c.rules
}

# Create startup script for DHT22
ROOTFS_POSTPROCESS_COMMAND += "add_dht22_startup;"

add_dht22_startup() {
    mkdir -p ${IMAGE_ROOTFS}/root/scripts
    
    # Create DHT22 reader script
    cat > ${IMAGE_ROOTFS}/root/scripts/read_dht22.sh << 'EOF'
#!/bin/bash
# DHT22 Reader over I2C
# Assumes DHT22 is on I2C address 0x40
# Uses i2cget to read humidity and temperature

I2C_BUS=1
I2C_ADDR=0x40

while true; do
    # Read 4 bytes from DHT22
    DATA=$(i2cget -y $I2C_BUS $I2C_ADDR 0x00 w)
    sleep 2
    echo "DHT22 Reading: $DATA"
done
EOF
    
    chmod +x ${IMAGE_ROOTFS}/root/scripts/read_dht22.sh
    
    # Create systemd service for DHT22
    cat > ${IMAGE_ROOTFS}/etc/systemd/system/dht22-reader.service << 'EOF'
[Unit]
Description=DHT22 I2C Sensor Reader
After=multi-user.target

[Service]
Type=simple
ExecStart=/root/scripts/read_dht22.sh
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

    # Enable service
    mkdir -p ${IMAGE_ROOTFS}/etc/systemd/system/multi-user.target.wants/
    ln -sf /etc/systemd/system/dht22-reader.service ${IMAGE_ROOTFS}/etc/systemd/system/multi-user.target.wants/dht22-reader.service
}

# Create boot message with IP info
ROOTFS_POSTPROCESS_COMMAND += "add_boot_message;"

add_boot_message() {
    cat > ${IMAGE_ROOTFS}/etc/profile.d/welcome.sh << 'EOF'
#!/bin/sh
echo "========================================="
echo "  my-custom-image - Raspberry Pi 4B+"
echo "========================================="
echo "I2C Devices:"
i2cdetect -y 1 2>/dev/null || echo "  No I2C devices found"
echo ""
echo "SSH Access: ssh root@$(hostname -I | awk '{print $1}')"
echo "========================================="
EOF
    chmod +x ${IMAGE_ROOTFS}/etc/profile.d/welcome.sh
}
```

3. Create HelloWorld C++ Application

File: meta-custom/recipes-apps/helloworld/files/helloworld.cpp

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <chrono>
#include <thread>

int main() {
    std::cout << "\n========================================" << std::endl;
    std::cout << "  Hello World from Raspberry Pi 4B+!" << std::endl;
    std::cout << "  Custom Yocto Image - Bare Metal Style" << std::endl;
    std::cout << "========================================\n" << std::endl;
    
    // Print I2C bus information
    std::ifstream cpuinfo("/proc/cpuinfo");
    std::string line;
    std::cout << "System Information:" << std::endl;
    while (std::getline(cpuinfo, line)) {
        if (line.find("Model") != std::string::npos ||
            line.find("Revision") != std::string::npos) {
            std::cout << "  " << line << std::endl;
        }
    }
    
    // Check I2C devices
    std::cout << "\nChecking I2C bus..." << std::endl;
    std::ifstream i2c_dev("/dev/i2c-1");
    if (i2c_dev.good()) {
        std::cout << "  ✓ I2C device /dev/i2c-1 found" << std::endl;
    } else {
        std::cout << "  ✗ I2C device not found" << std::endl;
    }
    
    // Network info
    std::cout << "\nNetwork Interfaces:" << std::endl;
    std::ifstream netfile("/proc/net/dev");
    std::string netline;
    while (std::getline(netfile, netline)) {
        if (netline.find("eth0") != std::string::npos) {
            std::cout << "  " << netline.substr(0, netline.find(":")) << " - Ethernet active" << std::endl;
        }
    }
    
    std::cout << "\nHelloWorld running successfully!" << std::endl;
    return 0;
}
```

File: meta-custom/recipes-apps/helloworld/files/CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
project(helloworld)

set(CMAKE_CXX_STANDARD 11)

add_executable(helloworld helloworld.cpp)

install(TARGETS helloworld RUNTIME DESTINATION bin)
```

File: meta-custom/recipes-apps/helloworld/helloworld.bb

```bitbake
SUMMARY = "HelloWorld C++ Application"
DESCRIPTION = "Simple C++ app demonstrating custom Yocto build"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://helloworld.cpp file://CMakeLists.txt"

S = "${WORKDIR}"

inherit cmake

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/helloworld ${D}${bindir}/
}

RDEPENDS:${PN} = "libstdc++"
```

4. Create I2C Device Tree Overlay for DHT22

File: meta-custom/recipes-kernel/device-tree/files/dht22-overlay.dts

```dts
/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2711";  // For RPi 4B+
    
    fragment@0 {
        target = <&i2c1>;
        __overlay__ {
            status = "okay";
            clock-frequency = <100000>;
            
            // DHT22 sensor on I2C address 0x40
            dht22: dht22@40 {
                compatible = "aosong,dht22";
                reg = <0x40>;
                status = "okay";
            };
        };
    };
    
    fragment@1 {
        target = <&gpio>;
        __overlay__ {
            dht22_pins: dht22_pins {
                brcm,pins = <2 3>;  // I2C pins
                brcm,function = <4>;  // ALT0 for I2C
            };
        };
    };
};
```

File: meta-custom/recipes-kernel/device-tree/dht22-overlay.bb

```bitbake
SUMMARY = "Device Tree Overlay for DHT22 I2C Sensor"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/GPL-2.0-only;md5=801f80980d171dd6425610833a22dbe6"

inherit devicetree

COMPATIBLE_MACHINE = "raspberrypi4-64"

SRC_URI = "file://dht22-overlay.dts"

S = "${WORKDIR}"

do_compile() {
    dtc -@ -I dts -O dtb -o dht22-overlay.dtbo ${S}/dht22-overlay.dts
}

do_install() {
    install -d ${D}/boot/overlays
    install -m 0644 ${S}/dht22-overlay.dtbo ${D}/boot/overlays/
}

FILES:${PN} = "/boot/overlays/*.dtbo"

RDEPENDS:${PN} = "kernel-devicetree"
```

5. Update Local Configuration

Add to conf/local.conf in your build directory:

```bash
cd ~/yocto-rpi4/poky/build-rpi4
nano conf/local.conf
```

Add these lines:

```bash
# Build my-custom-image
IMAGE_INSTALL:append = " helloworld dht22-overlay"

# Enable I2C in kernel
RPI_EXTRA_CONFIG:append = " dtparam=i2c_arm=on\n"

# Kernel modules for I2C
KERNEL_MODULE_AUTOLOAD:append = " i2c-dev i2c-bcm2835"

# Device tree overlays
KERNEL_DEVICETREE:append = " overlays/dht22-overlay.dtbo"

# I2C tools
IMAGE_INSTALL:append = " i2c-tools python3-smbus"
```

6. Build the Custom Image

```bash
cd ~/yocto-rpi4/poky
source oe-init-build-env build-rpi4

# Build your custom image
bitbake my-custom-image
```

7. Verify Build Output

```bash
ls -la tmp/deploy/images/raspberrypi4-64/my-custom-image-raspberrypi4-64.wic*
```

8. Flash and Test

```bash
# Flash to SD card
cd tmp/deploy/images/raspberrypi4-64/
sudo dd if=my-custom-image-raspberrypi4-64.wic of=/dev/sdX bs=4M status=progress
sync
```

9. On Raspberry Pi 4B+ Boot

After booting, you'll see:

```bash
# SSH into Pi
ssh root@<pi_ip>

# Should see welcome message with I2C detection

# Run HelloWorld
helloworld

# Check I2C devices (DHT22 should be at 0x40)
i2cdetect -y 1

# Read DHT22 sensor
/root/scripts/read_dht22.sh

# Check if overlay is loaded
cat /proc/device-tree/chosen/overlay/dht22-overlay/status
```

10. C++ DHT22 Reader Application (Optional - More Advanced)

File: meta-custom/recipes-apps/dht22reader/files/dht22reader.cpp

```cpp
#include <iostream>
#include <fcntl.h>
#include <unistd.h>
#include <linux/i2c-dev.h>
#include <sys/ioctl.h>
#include <chrono>
#include <thread>

class DHT22Reader {
private:
    int i2c_fd;
    int device_address;
    
public:
    DHT22Reader(int bus = 1, int address = 0x40) {
        std::string bus_path = "/dev/i2c-" + std::to_string(bus);
        i2c_fd = open(bus_path.c_str(), O_RDWR);
        device_address = address;
        
        if (i2c_fd < 0) {
            throw std::runtime_error("Failed to open I2C bus");
        }
        
        if (ioctl(i2c_fd, I2C_SLAVE, device_address) < 0) {
            throw std::runtime_error("Failed to acquire I2C bus access");
        }
    }
    
    ~DHT22Reader() {
        if (i2c_fd >= 0) close(i2c_fd);
    }
    
    void readSensor() {
        // Send measurement command
        unsigned char cmd = 0x00;
        if (write(i2c_fd, &cmd, 1) != 1) {
            std::cerr << "Failed to write to DHT22" << std::endl;
            return;
        }
        
        // Wait for measurement (DHT22 takes ~2 seconds)
        std::this_thread::sleep_for(std::chrono::milliseconds(2000));
        
        // Read 4 bytes of data
        unsigned char data[4];
        if (read(i2c_fd, data, 4) != 4) {
            std::cerr << "Failed to read from DHT22" << std::endl;
            return;
        }
        
        // Convert data (DHT22 protocol)
        uint16_t humidity_raw = (data[0] << 8) | data[1];
        uint16_t temperature_raw = (data[2] << 8) | data[3];
        
        float humidity = humidity_raw * 0.1;
        float temperature = temperature_raw * 0.1;
        
        std::cout << "Temperature: " << temperature << "°C, Humidity: " << humidity << "%" << std::endl;
    }
};

int main() {
    std::cout << "DHT22 Sensor Reader on Raspberry Pi 4B+" << std::endl;
    std::cout << "========================================" << std::endl;
    
    try {
        DHT22Reader sensor(1, 0x40);
        
        while (true) {
            sensor.readSensor();
            std::this_thread::sleep_for(std::chrono::seconds(5));
        }
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
        return 1;
    }
    
    return 0;
}
```

Summary

Your custom image my-custom-image now includes:

✅ SSH over Ethernet (always enabled with empty root password)
✅ HelloWorld C++ app (runs on boot or manually)
✅ DHT22 I2C sensor support with device tree overlay
✅ All Yocto metadata in poky/ directory
✅ Bare minimal - only essential features included

Build command:

```bash
bitbake my-custom-image
```

On Pi boot:

```bash
ssh root@<ip>  # No password
helloworld      # Run C++ app
i2cdetect -y 1  # Find DHT22 at 0x40
```