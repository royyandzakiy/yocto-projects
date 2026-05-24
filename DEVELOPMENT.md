- sstate
- preempt rt
- ota handling dual switching: rauc
- secure boot & chain of trust
- tfm if possible

- esp32 flasher
- zephyr flasher

- on rpi4b with uart

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
