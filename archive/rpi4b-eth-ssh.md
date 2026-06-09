Here's the modified guide for Raspberry Pi 4B+ with WSL2 setup and SSH over Ethernet:

---

Set Up Yocto for Raspberry Pi 4B+ (WSL2)

Prerequisites for WSL2

First, ensure your WSL2 environment has the necessary packages:

```bash
sudo apt-get update
sudo apt-get install gawk wget git-core diffstat unzip \
texinfo gcc-multilib build-essential chrpath zstd \
libssl-dev qemu-system-arm qemu-user-static u-boot-tools
```

WSL2 Network Setup for Ethernet Passthrough

To ensure SSH works over Ethernet from WSL2:

```powershell
# In Windows PowerShell (Admin)
# Bridge your Ethernet to WSL virtual switch
Get-NetAdapter  # Find your Ethernet adapter name
Set-VMSwitch -Name "WSL" -AllowNetLbfoTeams $true
```

Directory Structure (All metadata inside poky/)

```bash
mkdir -p ~/yocto-rpi4
cd ~/yocto-rpi4

# Clone everything into poky/
git clone https://github.com/yoctoproject/poky.git
cd poky

# Clone other layers INSIDE poky/
git clone https://github.com/agherzan/meta-raspberrypi
git clone https://github.com/openembedded/meta-openembedded.git
```

Your structure:

```
poky/
├── meta/
├── meta-poky/
├── meta-yocto-bsp/
├── meta-raspberrypi/    # Inside poky/
├── meta-openembedded/   # Inside poky/
│   ├── meta-oe/
│   ├── meta-python/
│   └── .../
```

Initialize Build Environment

```bash
cd ~/yocto-rpi4/poky
source oe-init-build-env build-rpi4
```

Configure for Raspberry Pi 4B+

Edit conf/local.conf:

```bash
nano conf/local.conf
```

Add/modify these lines:

```bash
# Target hardware
MACHINE = "raspberrypi4-64"

# Enable Ethernet and SSH
ENABLE_UART = "1"
ENABLE_I2C = "1"
ENABLE_SPI = "1"

# SSH configuration for Ethernet connection
EXTRA_IMAGE_FEATURES = "ssh-server-dropbear allow-empty-password empty-root-password allow-root-login debug-tweaks"

# Network tools for Ethernet
IMAGE_INSTALL:append = " net-tools iptables dhcp-client"

# Kernel configuration for Ethernet
KERNEL_MODULE_AUTOLOAD:append = " dwc2 g_ether"
KERNEL_MODULE_AUTOLOAD:append_rpi = " smsc95xx"

# Accept license for Raspberry Pi firmware
LICENSE_FLAGS_ACCEPTED = "synaptics-killswitch commercial"

# Optional: Speed up build (adjust to your system)
BB_NUMBER_THREADS = "4"
PARALLEL_MAKE = "-j 4"

# Static IP for Ethernet (optional - or use DHCP)
# Append to local.conf:
# # Static IP config
# IMAGE_INSTALL:append = " systemd-networkd"
# 
# # Create network config file
# write_env_file:append() {
#     cat > ${IMAGE_ROOTFS}/etc/systemd/network/20-ethernet.network << EOF
# [Match]
# Name=eth0
# 
# [Network]
# DHCP=yes
# EOF
# }
```

Configure Layers (conf/bblayers.conf)

Run these commands to set up layers properly:

```bash
bitbake-layers add-layer ../meta-raspberrypi
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-openembedded/meta-networking
```

Or manually edit conf/bblayers.conf:

```bash
nano conf/bblayers.conf
```

Ensure it looks like:

```bash
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  ${TOPDIR}/../meta \
  ${TOPDIR}/../meta-poky \
  ${TOPDIR}/../meta-yocto-bsp \
  ${TOPDIR}/../meta-raspberrypi \
  ${TOPDIR}/../meta-openembedded/meta-oe \
  ${TOPDIR}/../meta-openembedded/meta-python \
  ${TOPDIR}/../meta-openembedded/meta-networking \
  "
```

WSL2 Ethernet Bridge Script

Create a script to forward Ethernet to WSL2 (saved as ~/bridge_eth_to_wsl.sh):

```bash
#!/bin/bash
# Run this in WSL2 when Ethernet is connected

# Get WSL2 IP
WSL_IP=$(hostname -I | awk '{print $1}')

# Get Windows Ethernet IP (from Windows host)
WIN_ETH_IP=$(ip route | grep default | awk '{print $3}')

echo "WSL IP: $WSL_IP"
echo "Windows Ethernet Gateway: $WIN_ETH_IP"

# Add route for Ethernet subnet
sudo ip route add 192.168.1.0/24 via $WIN_ETH_IP dev eth0 2>/dev/null || true
```

Build the Image

```bash
cd ~/yocto-rpi4/poky/build-rpi4
bitbake core-image-base
```

Other options:

· core-image-minimal - Very minimal
· core-image-base - Good for headless with Ethernet
· rpi-test-image - Full Raspberry Pi test image

Flash to SD Card

After successful build:

```bash
cd tmp/deploy/images/raspberrypi4-64/

# Find your SD card (WSL2 - use Windows drive mapping)
lsblk  # Typically /dev/sdb or /dev/sdc in WSL2

# Flash the image
sudo dd if=core-image-base-raspberrypi4-64.wic of=/dev/sdX bs=4M status=progress
sync
```

Note: In WSL2, you may need to use Windows tools to flash the SD card, or install WSL2 kernel with USB support.

Connect via SSH Over Ethernet

1. On first boot (Raspberry Pi connected via Ethernet):
   · Router assigns IP via DHCP
   · Root login allowed with empty password
2. Find your Pi's IP:
   ```bash
   # From Windows (PowerShell)
   arp -a | findstr "b8:27:eb"  # Raspberry Pi MAC prefix
   
   # Or scan network
   nmap -sn 192.168.1.0/24  # Adjust subnet
   ```
3. SSH connection:
   ```bash
   ssh root@<raspberry_pi_ip>
   # Password: (empty, just press enter)
   ```

For static Ethernet connection on Pi side:

Add to local.conf before building:

```bash
# Static IP configuration for eth0
IMAGE_INSTALL:append = " systemd-networkd"

# Create network configuration in rootfs
ROOTFS_POSTPROCESS_COMMAND += "configure_static_eth;"
configure_static_eth() {
    mkdir -p ${IMAGE_ROOTFS}/etc/systemd/network/
    cat > ${IMAGE_ROOTFS}/etc/systemd/network/20-eth0.network << EOF
[Match]
Name=eth0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=8.8.8.8
EOF
}
```

Troubleshooting WSL2 SSH Access

If you can't SSH from WSL2 to Pi:

```bash
# In WSL2 - check if you can reach Pi
ping <pi_ip>

# If no route, check Windows firewall
# In Windows PowerShell (Admin):
New-NetFirewallRule -DisplayName "WSL-Allow-SSH" -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow

# Forward port from Windows to WSL (if needed)
# In Windows PowerShell (Admin):
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 connectport=22 connectaddress=<wsl_ip>
```

Quick Start Summary

```bash
# 1. Setup directories
mkdir -p ~/yocto-rpi4 && cd ~/yocto-rpi4
git clone https://github.com/yoctoproject/poky.git
cd poky
git clone https://github.com/agherzan/meta-raspberrypi
git clone https://github.com/openembedded/meta-openembedded.git

# 2. Build environment
source oe-init-build-env build-rpi4

# 3. Configure
echo 'MACHINE = "raspberrypi4-64"' >> conf/local.conf
echo 'EXTRA_IMAGE_FEATURES = "ssh-server-dropbear allow-empty-password empty-root-password allow-root-login"' >> conf/local.conf
echo 'LICENSE_FLAGS_ACCEPTED = "synaptics-killswitch"' >> conf/local.conf

# 4. Add layers
bitbake-layers add-layer ../meta-raspberrypi
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-python

# 5. Build
bitbake core-image-base
```

This configuration ensures the Raspberry Pi 4B+ will always have Ethernet available and SSH accessible from your WSL2 setup.