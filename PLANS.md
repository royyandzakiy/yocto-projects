## To Do
- [qemuarm64-custom-layer](qemuarm64-custom-layer.md)
- [qemuarm64-dts-led](qemuarm64-dts-led.md)
- [rpi-esp32-flasher](rpi-esp32-flasher.md)
- [qemuarm64-build-preemptrt](qemuarm64-build-preemptrt.md)
- [qemuarm64-build-ruac](qemuarm64-build-ruac.md)

## Done
- [qemuarm64-build-1](qemuarm64-build-1.md)
- [qemuarm64-build-2](qemuarm64-build-2.md)
- [qemuarm64-build-kas-1 (fail)](qemuarm64-build-kas-1.md)


---

- sstate
- preempt rt
- ota handling dual switching: rauc
- secure boot & chain of trust
- tfa

- esp32 flasher
- zephyr flasher

- on rpi4b with uart
- qt
- cpp23
- vcpkg

- custom board and dts
- handle sbom

- kas

---

## qemu yocto image
<TBD>

## Rpi 4B yocto image
- continue by flashing into sdcard > test if works fine on real raspi
	- continue here `royya@tuff16:~/project-coding/yocto/poky/build`
	- consider try install normal raspi image to check if sd & raspi work fine

```bash
royya@tuff16:~/project-coding/yocto/poky/build$ sudo apt update && sudo apt install liblz4-tool
royya@tuff16:~/project-coding/yocto/poky/build$ nano conf/local.conf
git config --global http.postBuffer
royya@tuff16:~/project-coding/yocto/poky/build$ bitbake -c fetchall rpi-test-image
royya@tuff16:~/project-coding/yocto/poky/build$ bitbake rpi-test-image --runall=fetch && bitbake rpi-test-image
```
