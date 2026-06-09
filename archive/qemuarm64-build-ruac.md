Here is the concise, step-by-step workflow to get RAUC running on `qemuarm64` using `runqemu`.

## 1. Setup the Environment

Clone the community layer that contains the pre-configured QEMU recipes and add it to your setup:

```bash
cd poky
git clone https://github.com/rauc/meta-rauc-community.git
bitbake-layers add-layer meta-rauc-community/meta-rauc-qemusystem

```

---

## 2. Configure `local.conf`

Append these lines to your `conf/local.conf` to set the machine target, inject the public keyring, and enable bundle generation:

```text
MACHINE = "qemuarm64"
DISTRO_FEATURES:append = " rauc"
IMAGE_INSTALL:append = " rauc rauc-hawkbit-updater"

# Use the community layer's pre-made dev certificates
RAUC_KEYRING_FILE = "${TOPDIR}/../meta-rauc-community/meta-rauc-qemusystem/files/ca.crt"
RAUC_KEY_FILE = "${TOPDIR}/../meta-rauc-community/meta-rauc-qemusystem/files/development-1.key.pem"
RAUC_CERT_FILE = "${TOPDIR}/../meta-rauc-community/meta-rauc-qemusystem/files/development-1.cert.pem"

```

---

## 3. Build & Run Base Image

Compile the core reference image and launch it inside QEMU:

```bash
bitbake core-image-minimal
runqemu qemuarm64 nographic

```

*Log in as `root` (no password).*

---

## 4. Check Current Slot Status

Inside the running QEMU terminal, verify that RAUC recognizes your layout:

```bash
rauc status

```

> **What to look for:** It should show two slots (`slot0`, `slot1`), with one marked as `booted` and `active`.

---

## 5. Modify and Build the Update Bundle

Leave QEMU running. Open a **new terminal** on your host PC to build the update bundle:

```bash
# Optional: Change a file or bump a version to verify the change later
echo "IMAGE_VERSION = \"2.0\"" >> conf/local.conf

# Build the standalone update bundle (.raucb)
bitbake qemu-demo-bundle

```

---

## 6. Stream and Install Local Update

1. **On Host PC:** Serve the compiled bundle over a local python server from your deploy directory:
```bash
cd tmp/deploy/images/qemuarm64/
python3 -m http.server 8080

```



```

2. **Inside QEMU Terminal:** Trigger the update by pulling from the host's virtual IP gateway (`10.0.2.2`):
   ```bash
   rauc install http://10.0.2.2:8080/qemu-demo-bundle-qemuarm64.raucb

```

3. **Reboot:** Once it reports success, restart the VM:
```bash
reboot

```



```

The bootloader will automatically catch the swap flag and boot you into the secondary slot. Run `rauc status` again to verify the active slot has flipped.

```