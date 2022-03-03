# ubuntu-core-riscv64
Provides the tooling for building a UC20 Image on RISCV64.

## This is in no way supported by Canonical.


### Structure

This repository is segmented into several major pieces:

| Object             | Purpose                            |
|--------------------|------------------------------------|
| gadget/            | Gadget Snap                        |
| kernel/            | Kernel Snap                        |
| \*.json            | Model JSON                         |
| qemu-run           | Qemu Invocation                    |
| riscv64-components | Bits required to build Initrd Snap |


### Instructions

The high level instructions for this project are quite simple:
1) [Build a RISC-V 64-bit Ubuntu Core initrd snap](riscv64-components/README.md)
2) [Build a Kernel Snap using that initrd snap for RISC-V](kernel/README.md)
3) [Build a Gadget Snap for your particular board](gadget/README.md)
4) Create an Ubuntu Core Image using that Gadget & Kernel
5) Successfully boot Ubuntu Core on RISC-V

More detailed information can be found in the READMEs!

Specifically, for the model:

```
# Make relevant modifications to ubuntu-core-20-riscv64.json if needed
snapcraft create-key riscy-key
snapcraft register-key riscy-key
snap sign -k riscy-key > ubuntu-core-20-riscv64.model < ubuntu-core-20-riscv64.json

ubuntu-image snap ubuntu-core-20-riscv64.model \
    --snap gadget/virt_*_riscv64.snap \
    --snap riscv64-virt-kernel_*_riscv64.snap
```

From there, you should be able to launch a Qemu virtual machine using the
created Ubuntu Core image as a ROOTFS by simply doing:
`./qemu-run`


### Future work

Drop `kernel/sources/config` and use `kconfig` in `snapcraft.yaml`
    * Potentially just minify the `config` to make kernel builds faster...

Boot on real hardware (boards arriving in the next month!)

Explore u-boot in SPL mode


### Explain Key Files
```
#!/bin/sh -e

qemu-system-riscv64 \
    -M virt -m 2G -smp 1 -nographic \
    -bios gadget/prime/fw_payload.elf \
    -drive file=virt.img,format=raw,if=virtio \
    -object rng-random,filename=/dev/urandom,id=rng0 \
    -device virtio-rng-device,rng=rng0 \
    -device virtio-net-device,netdev=usernet \
    -netdev user,id=usernet,hostfwd=tcp::22222-:22
```

We use Qemu's virt board out of simplicity -- other boards should also be
usable, simply modify the kernel and u-boot configs to match.

We add in a bios line pointing at our payload for two reasons. First, we need
this low-level bit to start everything moving. Second, if a bios file is not
explicitly specified, Qemu will automatically use a predefined one. This one
will, of course, not be using our u-boot, and we won't boot into UC :)

We use a virtio device as the drive (booting from SD is not supported on the
virt machine). This is important to consider for things like having VIRTIO_FS
built into the kernel, as `modprobe` is not part of `busybox` in the initrd.

The RNG device is just for fun (and entropy is great).

Port forwarding for SSH access.


```
{
    "type": "model",
    "series": "16",
    "authority-id": "cHcxHFRxgHBseRyUpLXtu6amXHgPyFQc",
    "brand-id": "cHcxHFRxgHBseRyUpLXtu6amXHgPyFQc",
    "model": "ubuntu-core-20-riscv64",
    "architecture": "riscv64",
    "timestamp": "2022-02-18T21:50:41+00:00",
    "base": "core20",
    "grade": "dangerous",
    "snaps": [
        {
            "name": "virt",
            "type": "gadget"
        },
        {
            "name": "riscv64-virt-kernel",
            "type": "kernel"
        },
        {
            "name": "core20",
            "type": "base",
            "default-channel": "latest/edge",
            "id": "DLqre5XGLbDqg9jPtiAhRRjDuPVa5X1q"
        },
        {
            "name": "snapd",
            "type": "snapd",
            "default-channel": "latest/edge",
            "id": "PMrrV4ml8uWuEUDBT8dSGnKUYbevVhc4"
        }
    ]
}
```

`{authority,brand}-id` point at the official Snapcraft store for simplicity.

`grade` is dangerous because we have to sideload our kernel and gadget snaps.

`default-channel` and `id` are optional fields for snaps, and so we don't need
to include any garbage data here for our gadget or kernel snaps. Just make sure
the name match and you actually have a declared gadget and kernel.
