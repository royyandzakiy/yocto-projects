## Yocto Raspberry Pi 4 Build - README (Scarthgap)

### Build Environment
- WSL2 (Ubuntu/Debian)
- Target: Raspberry Pi 4B (64-bit) with UART serial console + Ethernet SSH
- Image: `core-image-base` (includes SSH)
- Yocto Version: **Scarthgap** (latest LTS)

---

### 1. Clone Repositories

```bash
mkdir -p ~/project-coding/yocto && cd ~/project-coding/yocto

# Scarthgap branch (2024 LTS)
git clone -b scarthgap git://git.yoctoproject.org/poky
git clone -b scarthgap git://git.yoctoproject.org/meta-raspberrypi
```

---

### 2. Setup Build Environment

```bash
cd poky
source oe-init-build-env ../build-rpi4
bitbake-layers add-layer ../../meta-raspberrypi
```

---

### 3. Configure `conf/local.conf`

```ini
MACHINE = "raspberrypi4-64"

# ===== UART Serial Console =====
ENABLE_UART = "1"
SERIAL_CONSOLES = "115200;serial0"
RPI_EXTRA_CONFIG += "dtparam=uart0_console"

# ===== Ethernet & SSH =====
ENABLE_DWC2_PERIPHERAL = "1"
IMAGE_FEATURES += "ssh-server-openssh"

# ===== WiFi Configuration =====
# Enable WiFi firmware and utilities
DISTRO_FEATURES:append = " wifi"
IMAGE_INSTALL:append = " iw wireless-tools wpa-supplicant"

# Auto-connect to WiFi on boot (replace with your SSID/password)
WIFI_SSID = "YourNetworkSSID"
WIFI_PASSWORD = "YourNetworkPassword"

# Create wpa_supplicant config automatically
write_wpa_supplicant_conf() {
    cat > ${IMAGE_ROOTFS}/etc/wpa_supplicant.conf << EOF
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=0
update_config=1
network={
    ssid="${WIFI_SSID}"
    psk="${WIFI_PASSWORD}"
    key_mgmt=WPA-PSK
}
EOF
}
ROOTFS_POSTPROCESS_COMMAND += "write_wpa_supplicant_conf;"

# Enable WiFi interface at boot
CORE_IMAGE_EXTRA_INSTALL += " \
    iw \
    wireless-tools \
    wpa-supplicant \
    dhcpcd \
"

# ===== Empty root password (development only) =====
EXTRA_IMAGE_FEATURES ?= "debug-tweaks"

# ===== Optional: Static IP for Ethernet/WiFi =====
# Uncomment to use static IP instead of DHCP
# ENABLE_STATIC_IP = "1"
# STATIC_IP = "192.168.1.100"
# STATIC_NETMASK = "255.255.255.0"
# STATIC_GATEWAY = "192.168.1.1"

# ===== Optional: Add apps =====
IMAGE_INSTALL:append = " i2c-tools vim htop python3"
```

---

### 4. Build Image

```bash
bitbake core-image-base
```

*(Takes 1-3 hours on first build - includes SSH packages)*

---

### 5. Locate & Copy Image

```bash
cd ../build-rpi4/tmp/deploy/images/raspberrypi4-64/
cp core-image-base-raspberrypi4-64.wic.bz2 /mnt/c/Users/YOUR_USER/Downloads/
```

---

### 6. Flash SD Card (Windows)

- Open **Balena Etcher**
- Select the `.wic.bz2` file
- Select SD card → **Flash**

---

### 7. Connect UART to PC

| USB-to-TTL | RPi GPIO |
|------------|----------|
| GND | Pin 6 |
| RX | Pin 8 (GPIO14/TXD) |
| TX | Pin 10 (GPIO15/RXD) |

**Serial settings:** `115200 baud, 8N1, no flow control`

---

### 8. Boot & Login

**Via UART (Console):**
- `screen /dev/ttyUSB0 115200` (Linux) or PuTTY (Windows)
- Login: `root` (no password)

**Via SSH (Ethernet):**
```bash
# Find RPi IP from serial console (run on RPi):
ip addr show eth0

# Or check your router's DHCP leases

# From your PC:
ssh root@<rpi-ip-address>
```

---

### Clean Build Commands

| Task | Command (from build-rpi4 directory) |
|------|--------------------------------------|
| Rebuild | `bitbake core-image-base` |
| Clean all | `bitbake -c cleanall core-image-base` |
| Remove build artifacts | `rm -rf tmp/` |
| Start fresh | `rm -rf ../build-rpi4 && source oe-init-build-env ../build-rpi4` |

---

### Add Packages to Image

Edit `conf/local.conf`:

```ini
IMAGE_INSTALL:append = " i2c-tools vim htop python3"
```

Then rebuild.

---

### Notes
- **Scarthgap** is the 2024 LTS release (supports through 2028)
- `core-image-base` includes networking, SSH, and basic utilities vs `core-image-minimal`
- First boot may take 30-60 seconds to generate SSH host keys
- Default SSH allows root login with empty password (dev only!)