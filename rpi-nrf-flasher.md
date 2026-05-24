To transform your Raspberry Pi 4B into a flashing station for a Nordic **nRF5340 Development Kit (nRF5340DK)**, the strategy changes slightly compared to the ESP32.

Nordic chips rely on **SWD (Serial Wire Debug)** rather than a simple UART bootloader. To flash them from a Raspberry Pi, you need a hardware abstraction utility that can control the SWD pins. You have two excellent options:

1. **openocd (Open On-Chip Debugger):** Fully open-source, uses the Raspberry Pi's GPIO pins directly as a programmer (no extra hardware required).

2. **nrf-command-line-tools (nrfjprog):** Nordic's official utility. This is ideal if you are connecting the Raspberry Pi to the nRF5340DK via a standard USB cable, as the DK has an onboard Segger J-Link debugger.

Here is how to add both approaches to your Yocto layers.

## 1. Add Flash Utilities and Dependencies to local.conf

Open conf/local.conf and add the packages needed to talk to the J-Link debugger (via USB) or directly over GPIO (via SWD):

```bash

# Add OpenOCD for direct GPIO-to-SWD flashing

IMAGE_INSTALL:append = " openocd"



# Add USB communication utilities if connecting via USB cable

IMAGE_INSTALL:append = " libusb1 usbutils"



# Ensure curl/wget and bash are available for your script

IMAGE_INSTALL:append = " curl bash"



```

## 2. Update Your Custom Script Layer

Let's expand or create a new recipe variant for your download and flash runner script.

### Step A: Create the Script File

Create or modify a script at meta-raspberrypi/recipes-utils/fw-flasher/files/flash-nrf.sh.

Depending on your hardware layout, choose **one** of the two methods below inside your script:

#### Method Option 1: Over USB (Using OpenOCD targeting the onboard J-Link)

If the Pi is connected to the nRF5340DK via a USB cable:

```bash

#!/bin/bash

# flash-nrf.sh (USB J-Link Method)

BINARY_URL="https://your-server.com/api/firmware/latest_merged.hex"

SAVE_PATH="/tmp/zephyr_latest.hex"



echo "Downloading latest Zephyr hex..."

curl -L -o $SAVE_PATH $BINARY_URL



if [ $? -eq 0 ] && [ -f "$SAVE_PATH" ]; then

    echo "Download complete. Flashing nRF5340 via OpenOCD..."

    # nRF5340 is a dual-core chip; we target the application core (cpunet can be targeted similarly)

    openocd -f interface/jlink.cfg -f target/nrf53.cfg -c "program $SAVE_PATH verify reset exit"

else

    echo "Download failed."

    exit 1

fi



```

#### Method Option 2: Over Pi GPIO Pins (Direct SWD Flashing)

If you are wiring the Raspberry Pi GPIO pins directly to the nRF5340DK SWD debug header:

```bash

#!/bin/bash

# flash-nrf.sh (Native GPIO SWD Method)

SAVE_PATH="/tmp/zephyr_latest.hex"



# OpenOCD command using the Pi's internal peripheral driver for SWD pins

# Requires an openocd configuration mapping your chosen GPIO lines to SWD_CLK and SWD_DIO

openocd -f interface/raspberrypi-native.cfg -f target/nrf53.cfg -c "program $SAVE_PATH verify reset exit"



```

### Step B: Update the BitBake Recipe File

Update your meta-raspberrypi/recipes-utils/fw-flasher/fw-flasher_1.0.bb recipe to include the new script:

```bitbake

SUMMARY = "Custom firmware downloader and nRF5340 flasher script"

SECTION = "utils"

LICENSE = "MIT"

LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"



SRC_URI = " \

    file://flash-fw.sh \

    file://flash-nrf.sh \

"



S = "${WORKDIR}"



RDEPENDS:${PN} = "bash curl openocd libusb1"



do_install() {

    install -d ${D}${usrbin}

    install -m 0755 flash-fw.sh ${D}${usrbin}/flash-fw

    install -m 0755 flash-nrf.sh ${D}${usrbin}/flash-nrf

}



FILES:${PN} = "${usrbin}/flash-fw ${usrbin}/flash-nrf"



```

## 3. Include Your Recipe and Build

Ensure fw-flasher is appended to your image manifest inside conf/local.conf:

```bash

IMAGE_INSTALL:append = " fw-flasher"



```

Rebuild your Yocto distribution:

```bash

cd ~/project-coding/yocto/build

bitbake core-image-sato



```

## 💡 A Vital Note on Zephyr Binaries for nRF5340

Because the nRF5340 is a **dual-core** chip (Application Core and Network Core), your build output system matters:

* **Always Flash the Hex File:** Do not use the raw .bin file. Zephyr generates a **zephyr.hex** or a **merged.hex** file. Hex files contain explicit memory address configurations telling OpenOCD exactly where the payload belongs.

* **The merged.hex File:** If your application utilizes the Network Core (e.g., for Bluetooth LE), Zephyr automatically builds both images and merges them into merged.hex. Ensure your online script pulls this file so both cores are updated simultaneously during production flashing.