# container2boot: Simple and Easy Bootable Containers

> Convert a podman container into a bootable OS image… without privileges!

There are hundreds of ways to build Linux images. Mature tools for building images include:

* [`livecd-tools`](https://github.com/livecd-tools/livecd-tools): for building DVD/ISO live images

* [`mkosi`](https://github.com/systemd/mkosi/): a more modern builder for generating images from scratch with [systemd-repart](https://man.archlinux.org/man/systemd-repart.8.en).

* [`d2vm`](https://github.com/linka-cloud/d2vm): converts containers to bootable images, but requires privileged Docker containers

* [`cosa`](https://github.com/coreos/coreos-assembler): the CoreOS Assembler, for atomically-upgraded Fedora CoreOS images. This tool will [eventually](https://github.com/coreos/fedora-coreos-tracker/issues/1151) support generating OS images from containers.

If you're looking for a mature and/or well-supported image builder, check out one of the above products.

This project is a **proof-of-concept** that operates slightly differently than any of the above. Goals are:

* ✔ Up-convert an existing container into a bare-metal bootable system…

* ✔ …that boots like a LiveCD: i.e., as a non-persistent system with no changes saved…

* ✔ …using only **unprivileged containers** that operate without sudo rights or particular requirements on the host. This last requirement is the most difficult.

Disk images for VMs or bare-metal systems can be difficult to work with. They usually require privileged kernel loop devices or virtual machine software like [guestfish](https://libguestfs.org/guestfish.1.html) to access. Deriving a new VM from a "base" image often involves booting the image and modifying it online. In contrast, container technologies make these things easy.

Containers are a natural storage format for machine images because they take care of mapping UIDs and GIDs to the host system. With [podman](https://podman.io/), even unprivileged users can build and start containers. Unlike disk images, it is easy to extend an existing container image with additional software. Container registries are also a convenient storage and transport mechanism.

So, instead of using any of the existing image builders, let's turn a container into a bootable, working system!

To run this example, you need:

* `podman >= 4.3` with `buildah`
* access to public container registries
* access to the AlmaLinux and Fedora DNF repositories
* (optional) a VM like libvirt and/or qemu to test the image

## Example machine

An ordinary base image, like [`docker.io/library/almalinux`](https://hub.docker.com/_/almalinux), doesn't have nearly enough software to boot. For one thing, it's missing a kernel and bootloaders. Container images also do not come with essential server software like `sshd` or `NetworkManager`; they don't need them. Since containers rely on the host kernel to enforce SELinux policy, they don't have their own policy files either. Let's fix this by creating a full AlmaLinux server installation with our `machine` Dockerfile.

```bash
# --secret …     sets a passwd so you can log in

DOCKER_BUILDKIT=1 podman build \
  -f Dockerfile.machine \
  --secret id=chpass,src=./democreds \
  --tag localhost/container2boot/machine:latest
```

See [Dockerfile.machine](./Dockerfile.machine). These instructions:

1. Add a kernel and other required userspace programs
2. Add some initrd programs that are useful for a LiveCD-style boot
3. Create users from the [`democreds`](./democreds) file.

The resulting image is a complete AlmaLinux server system. It is *very* heavyweight—about 2.25 GB as of this writing. Feel free to start it and inspect it.

```bash
podman run --rm -it localhost/container2boot/machine:latest
```

We'll make this image bootable in a moment.

## Building Basic Images

[`Dockerfile.rootbuilder`](./Dockerfile.rootbuilder) is a Fedora 39 (or higher) image that ingests a ready-to-boot rootfs and transforms it into a bootable image. First, build it:

```bash
DOCKER_BUILDKIT=1 podman build \
  -f Dockerfile.rootbuilder \
  --tag localhost/container2boot/rootbuilder:latest
```

The above image consists mainly of a shell script, [`rootbuilder`](rootbuilder/usr/local/bin/rootbuilder), which builds the bootable disk image. When executed, this container will take the image mounted at `/rootfs` and output a kernel image, initrd, and a compressed rootfs to the volume mounted at `/out`.

For example,

```bash
# --mount …    picks the image to boot-ify

podman run --rm -it \
  --mount=type=image,source=localhost/container2boot/machine:latest,target=/rootfs \
  --volume ./out:/out:rw \
  localhost/container2boot/rootbuilder:latest
```

Which will write a `vmlinuz`, `initrd.img`, and `squashfs.img` file to `./out/base`. The squashfs image is generated from the mounted container with [`gensquashfs`](https://github.com/AgentD/squashfs-tools-ng).

The basic output is only bootable in the strictest sense: it only works as a PXE or netboot system. We'll generate a more complete disk image in a moment, but first let's try to start it in qemu.

Start a web server to host the rootfs image:

```bash
cd out/base
python -m http.server 8082
```

And run qemu in another terminal:

```bash
qemu-system-x86_64 \
  -nographic \
  -kernel out/base/vmlinuz \
  -initrd out/base/initrd.img \
  -enable-kvm \
  -machine accel=kvm -cpu host \
  -m 4G \
  -netdev user,id=mynet0,net=192.168.76.0/24 \
  -device virtio-net-pci,netdev=mynet0 \
  -append 'console=ttyS0 root=live:http://192.168.76.2:8082/squashfs.img rd.neednet=1 rd.live.image rd.live.overlay.readonly enforcing=1'
```

Use Ctrl-a x to kill the VM.

Thanks to the SELinux labeling capability of `gensquashfs`, the VM boots just fine in SELinux enforcing mode.

The network boot functionality depends on dracut's [`livenet`](https://github.com/dracutdevs/dracut/tree/master/modules.d/90livenet) module to fetch the `root=` filesystem over the network. Then, [`dmsquash-live`](https://github.com/dracutdevs/dracut/tree/master/modules.d/90dmsquash-live) mounts it. The squashfs is mounted as `/` with a tmpfs writable layer. Like a LiveCD, you can change any file, but these changes will be lost on a reboot.

This example generates an AlmaLinux 9 system. It may work on other operating systems that have kernel support for squashfs and use dracut as their initrd. More work is required to adapt it to Debian.
