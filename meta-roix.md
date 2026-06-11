## Yocto meta-roix Layer Build - README (Scarthgap)

### Build Environment
- WSL2 (Ubuntu/Debian)
- Target: Raspberry Pi 4B (64-bit) with UART serial console + Ethernet SSH
- Image: `roix-base` (custom layer, includes SSH + systemd-networkd)
- Layer: `meta-roix` | Distro: `roix` (Roy Linux) | Machine: `raspberrypi4-64-roix`
- Yocto Version: **Scarthgap** (latest LTS)

---

### 1. Clone Repositories

```bash
mkdir -p ~/project-coding/yocto && cd ~/project-coding/yocto

git clone -b scarthgap https://github.com/yoctoproject/poky
git clone -b scarthgap https://github.com/agherzan/meta-raspberrypi        meta-layers/meta-raspberrypi
git clone -b scarthgap https://github.com/openembedded/meta-openembedded   meta-layers/meta-openembedded
# place meta-roix at meta-layers/meta-roix
```

---

### 2. Setup Build Environment

```bash
source poky/oe-init-build-env builds/build-roix
```

---

### 3. Configure `conf/bblayers.conf`

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
  /path/to/meta-layers/meta-roix \
  "
```

---

### 4. Configure `conf/local.conf`

```ini
MACHINE = "raspberrypi4-64-roix"
DISTRO  = "roix"

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

---

### 5. Build Image

```bash
bitbake roix-base
```

*(Takes 1-3 hours on first build)*

---

### 6. Locate Image

```bash
cd tmp/deploy/images/raspberrypi4-64-roix/

# Each build writes a new timestamped image; grab the latest:
ls -t *rootfs-*.wic.bz2 | head -1
```

The image you flash is `roix-base-raspberrypi4-64-roix.rootfs-<TIMESTAMP>.wic.bz2`,
with a matching `.wic.bmap` (block map) sitting next to it.

> The non-timestamped `roix-base-raspberrypi4-64-roix.rootfs.wic.bz2` is just a
> **symlink** to the latest build — `bzip2 -d` refuses to decompress it in place
> ("not a regular file"). Always operate on the timestamped file, or decompress
> to a new file with `bunzip2 -kc <img> > out.wic` (`-k` keep, `-c` to stdout).

---

### 7. Flash SD Card — directly from WSL2 (no Etcher)

WSL2 doesn't see USB block devices by default. Use **usbipd-win** to attach the
SD-card reader from Windows into WSL2, then write to it with `bmaptool` or `dd`.

**a) Attach the reader (Windows, admin PowerShell):**

```powershell
usbipd list                      # find the reader's BUSID (e.g. 2-4)
usbipd bind   --busid <BUSID>    # one-time, marks it shareable
usbipd attach --wsl --busid <BUSID>
```

**b) Identify the device (WSL2):**

```bash
lsblk -o NAME,SIZE,RM,TYPE,MOUNTPOINTS
```

The SD card is the **removable** disk (`RM 1`), e.g. `sde 29.2G`.

> ⚠️ **Target the whole disk (`/dev/sde`), never a partition (`/dev/sde1`).**
> And never your WSL system disk (the large one mounted at `/`). A wrong target
> wipes it. Device letters can change between sessions — re-check `lsblk` every time.

**c) Unmount anything auto-mounted, then flash:**

```bash
sudo umount /dev/sde1 2>/dev/null   # harmless if not mounted

IMG=$(ls -t *rootfs-*.wic.bz2 | head -1) && echo "Flashing: $IMG"
```

*Option A — `bmaptool` (recommended: skips empty blocks, verifies checksums):*

```bash
sudo apt install -y bmap-tools          # one-time
sudo bmaptool copy "$IMG" /dev/sde      # reads .bz2 + .bmap automatically
sudo sync
```

*Option B — `dd` (always available):*

```bash
bunzip2 -kc "$IMG" | sudo dd of=/dev/sde bs=4M conv=fsync iflag=fullblock status=progress
sudo sync
```

**d) Detach from WSL2 (Windows, admin PowerShell):**

```powershell
usbipd detach --busid <BUSID>
```

Then move the card to the Pi and boot.

---

### 8. Connect UART to PC

| USB-to-TTL | RPi GPIO |
|------------|----------|
| GND | Pin 6 |
| RX | Pin 8 (GPIO14/TXD) |
| TX | Pin 10 (GPIO15/RXD) |

**Serial settings:** `115200 baud, 8N1, no flow control`

---

### 9. Boot & Login

**Via UART (Console):**
- `screen /dev/ttyUSB0 115200` (Linux) or PuTTY (Windows)
- Login: `root` (no password)

**Via SSH (Ethernet):**
```bash
# Find RPi IP from serial console (run on RPi):
ip addr show eth0

# From your PC:
ssh root@<rpi-ip-address>
```

---

### Clean Build Commands

**Per-recipe cleans** (replace `roix-base` with any recipe). Escalating severity:

| Command | Removes | Keeps | Use when |
|---------|---------|-------|----------|
| `bitbake -c clean roix-base` | recipe work dir + its deploy output | sstate, downloads | redo a recipe — replays from sstate, fast |
| `bitbake -c cleansstate roix-base` | above **+ recipe's sstate cache** | downloads | force a genuine rebuild of that recipe |
| `bitbake -c cleanall roix-base` | above **+ recipe's downloaded source** | — | recipe broken / corrupt download |

**Whole-build cleanup** (run inside `builds/build-roix/`). Space lives in three dirs
— `tmp/` (all build output), `downloads/` (`DL_DIR`, upstream tarballs),
`sstate-cache/` (prebuilt task cache that makes rebuilds fast):

| Task | Command |
|------|---------|
| Reclaim space, keep fast rebuilds | `rm -rf tmp` *(sstate replays everything in minutes)* |
| Full nuke (next build = full 1–3 h + re-download) | `rm -rf tmp sstate-cache downloads` |
| Prune old flashed images | `rm deploy/images/*/...rootfs-<OLD_TIMESTAMP>.*` |
| Rebuild image | `bitbake roix-base` |
| Start completely fresh | `rm -rf builds/build-roix && source poky/oe-init-build-env builds/build-roix` |

> **Recommended:** `rm -rf tmp` only. With `sstate-cache` + `downloads` intact,
> a full image rebuild takes minutes, not hours. Delete those two only when you're
> done with the project or truly out of disk.

---

### Add Packages to Image

Edit `roix-base.bb`:

```bitbake
IMAGE_INSTALL:append = " python3 git vim htop"
```

Then rebuild.

---

### Notes
- **Scarthgap** is the 2024 LTS release (supports through 2028)
- `roix-base` extends `core-image-base` with SSH, UART, I2C, SPI, and eth0 DHCP via systemd-networkd
- `debug-tweaks` allows empty root password — remove for production
- First boot may take 30-60 seconds to generate SSH host keys
- `systemd-networkd` ships inside `systemd` — do not add it separately to `IMAGE_INSTALL`
