# meta-roy

Custom Yocto layer for Raspberry Pi 4 (64-bit) — headless base image with SSH, UART serial console, and eth0 DHCP.

**Branch:** `scarthgap`

---

## What it does

- Builds `roy-image-base` on top of `core-image-base`
- Enables OpenSSH server with root login (dev mode)
- Configures eth0 with DHCP via systemd-networkd (dropped into rootfs at build time)
- Enables UART on `serial0` via `ENABLE_UART = "1"` and `dtparam=uart0_console` — the proven working approach
- Enables I2C and SPI
- Uses `systemd` as init manager via a minimal custom distro (`roy`)

---

## Directory structure

```
meta-roy/
├── conf/
│   ├── layer.conf
│   ├── distro/
│   │   └── roy.conf                    # custom distro, inherits poky
│   └── machine/
│       └── raspberrypi4-64-roy.conf    # extends meta-raspberrypi machine conf
├── recipes-core/
│   └── images/
│       └── roy-image-base.bb           # the image recipe
└── README.md
```

---

## Dependencies

| Layer | Branch | Source |
|---|---|---|
| poky/meta | scarthgap | https://github.com/yoctoproject/poky |
| poky/meta-poky | scarthgap | same |
| poky/meta-yocto-bsp | scarthgap | same |
| meta-raspberrypi | scarthgap | https://github.com/agherzan/meta-raspberrypi |
| meta-openembedded/meta-oe | scarthgap | https://github.com/openembedded/meta-openembedded |
| meta-openembedded/meta-python | scarthgap | same |
| meta-openembedded/meta-networking | scarthgap | same |

---

## Setup

### 1. Clone dependencies

```bash
mkdir -p ~/project-coding/yocto && cd ~/project-coding/yocto

git clone -b scarthgap https://github.com/yoctoproject/poky
git clone -b scarthgap https://github.com/agherzan/meta-raspberrypi        meta-layers/meta-raspberrypi
git clone -b scarthgap https://github.com/openembedded/meta-openembedded   meta-layers/meta-openembedded
# place meta-roy at meta-layers/meta-roy
```

### 2. Init build environment

```bash
source poky/oe-init-build-env build-roy-image
```

### 3. Set bblayers.conf

```bitbake
POKY_BBLAYERS_CONF_VERSION = "2"
BBPATH = "${TOPDIR}"
BBFILES ?= ""
BBLAYERS ?= " \
  /path/to/poky/meta \
  /path/to/poky/meta-poky \
  /path/to/poky/meta-yocto-bsp \
  /path/to/meta-layers/meta-raspberrypi \
  /path/to/meta-layers/meta-openembedded/meta-oe \
  /path/to/meta-layers/meta-openembedded/meta-python \
  /path/to/meta-layers/meta-openembedded/meta-networking \
  /path/to/meta-layers/meta-roy \
  "
```

### 4. Set local.conf

```bitbake
MACHINE = "raspberrypi4-64-roy"
DISTRO  = "roy"

USER_CLASSES ?= "buildstats"
PATCHRESOLVE  = "noop"
BB_DISKMON_DIRS ??= "\
    STOPTASKS,${TMPDIR},1G,100K \
    STOPTASKS,${DL_DIR},1G,100K \
    STOPTASKS,${SSTATE_DIR},1G,100K \
    STOPTASKS,/tmp,100M,100K \
    HALT,${TMPDIR},100M,1K \
    HALT,${DL_DIR},100M,1K \
    HALT,${SSTATE_DIR},100M,1K \
    HALT,/tmp,10M,1K"
```

### 5. Build

```bash
bitbake roy-image-base
```

Output image lands in `tmp/deploy/images/raspberrypi4-64-roy/`.

---

## Hardware defaults

| Feature | Setting |
|---|---|
| Serial console | `serial0` at 115200 (maps to GPIO 14/15) |
| UART config | `ENABLE_UART = "1"` + `dtparam=uart0_console` |
| I2C | Enabled |
| SPI | Enabled |
| Ethernet | eth0, DHCP via systemd-networkd |
| SSH | OpenSSH, root login enabled (`debug-tweaks`) |
| Hostname | `roypi` |

> **Note:** `debug-tweaks` allows empty root password — fine for development,
> remove it for any production or exposed deployment.

---

## UART wiring

| USB-to-TTL | RPi GPIO |
|---|---|
| GND | Pin 6 |
| RX | Pin 8 (GPIO14 / TXD) |
| TX | Pin 10 (GPIO15 / RXD) |

**Serial settings:** `115200 baud, 8N1, no flow control`

Connect with `screen /dev/ttyUSB0 115200` (Linux) or PuTTY (Windows).

---

## SSH access

Find the RPi IP from the serial console after boot:

```bash
ip addr show eth0
```

Then from your PC:

```bash
ssh root@<rpi-ip-address>
# no password required (debug-tweaks)
```

---

## Flash SD card

Image is at `tmp/deploy/images/raspberrypi4-64-roy/roy-image-base-raspberrypi4-64-roy.wic.bz2`.

Use **Balena Etcher** or `dd`:

```bash
bunzip2 -c roy-image-base-raspberrypi4-64-roy.wic.bz2 | sudo dd of=/dev/sdX bs=4M status=progress
```

---

## Customisation

**Static IP** — replace the `[Network]` section in `roy-image-base.bb`'s `setup_eth0_dhcp`:

```ini
[Match]
Name=eth0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=1.1.1.1
```

**Add packages** — append to `IMAGE_INSTALL` in `roy-image-base.bb`:

```bitbake
IMAGE_INSTALL:append = " python3 git vim htop"
```

---

## Troubleshooting

**`Nothing RPROVIDES 'systemd-networkd'`** — not a standalone package in Yocto. The daemon ships inside `systemd`, pulled in automatically by `INIT_MANAGER = "systemd"`. Do not add it to `IMAGE_INSTALL`.

**Layer not found** — run `bitbake-layers show-layers` and confirm `meta-roy` appears at priority 10. If not, check the path in `bblayers.conf`.

**UART silent** — verify `enable_uart=1` and `dtparam=uart0_console` are present in `/boot/config.txt` on the flashed SD card. Also confirm TX/RX are not swapped on the USB-to-TTL adapter.

**First boot slow** — SSH host key generation on first boot can take 30–60 seconds before the login prompt appears.

---

## Maintainer

Roy — built with Yocto scarthgap / poky.