# Development

- properly etch the rpi image to rpi via balena etcher
```bash
royya@tuff16:~/project-coding/yocto/poky/build-rpi4/tmp/deploy/images/raspberrypi4-64$ ls **.wic**
core-image-base-raspberrypi4-64.rootfs-20260607224329.wic.bmap
core-image-base-raspberrypi4-64.rootfs-20260607224329.wic.bz2
core-image-base-raspberrypi4-64.rootfs.wic.bmap
core-image-base-raspberrypi4-64.rootfs.wic.bz2 <<<< this one
```

- currently still fails the etching, and cannot find in ethernet

```bash
C:\Users\royya>ping raspberrypi.local
Ping request could not find host raspberrypi.local. Please check the name and try again.
```

- either the build failed, etch failed, or ethernet failed. need to find out