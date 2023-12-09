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

This will write a directory `./out/esp` which contains a complete EFI System Partition (ESP) filesystem. The base system is installed in `./out/esp/container2boot` in the following three files:

* `vmlinuz`: the kernel

* `initrd.img`: the dracut-based initramfs / initrd, which is the
  initial userspace filesystem for the kernel

* `squashfs.img`: the root filesystem. This is generated from the
  mounted container with
  [`gensquashfs`](https://github.com/AgentD/squashfs-tools-ng).

We'll cover the other output files in the next section. For now, let's use qemu to "boot" these files directly without a bootloader like GRUB.

Start a web server to host the rootfs image:

```bash
cd out/esp/container2boot
python -m http.server 8082
```

And run qemu in another terminal:

```bash
qemu-system-x86_64 \
  -nographic \
  -kernel out/esp/container2boot/vmlinuz \
  -initrd out/esp/container2boot/initrd.img \
  -enable-kvm \
  -machine accel=kvm -cpu host \
  -m 4G \
  -netdev user,id=mynet0,net=192.168.76.0/24 \
  -device virtio-net-pci,netdev=mynet0 \
  -append 'console=ttyS0 root=live:http://192.168.76.2:8082/squashfs.img rd.neednet=1 rd.live.image rd.live.overlay.readonly enforcing=1'
```

Use Ctrl-a x to kill the VM, then `reset` your terminal.

Thanks to the SELinux labeling capability of `gensquashfs`, the VM boots just fine in SELinux enforcing mode.

The network boot functionality depends on dracut's [`livenet`](https://github.com/dracutdevs/dracut/tree/master/modules.d/90livenet) module to fetch the `root=` filesystem over the network. Then, [`dmsquash-live`](https://github.com/dracutdevs/dracut/tree/master/modules.d/90dmsquash-live) mounts it. The squashfs is mounted as `/` with a tmpfs writable layer. Like a LiveCD, you can change any file, but these changes will be lost on a reboot.

There are no containers or container-like things happening at boot! Instead, we've just converted the source container into a regular LiveCD system that isn't running podman or the like.

This example generates an AlmaLinux 9 system. It may work on other operating systems that have kernel support for squashfs and use dracut as their initrd. More work is required to adapt it to Debian.

## Disk Images

The rootbuilder image can also generate a complete GPT disk image

```bash
podman run --rm -it \
  --mount=type=image,source=localhost/container2boot/machine:latest,target=/rootfs \
  --volume ./out:/out:rw \
  localhost/container2boot/rootbuilder:latest \
  rootbuilder --diskimage
```

This generates a GPT disk image to `./out/disk.img` from the contents of `./out/esp`. The GPT image contains an ESP vfat filesystem with `PARTLABEL=C2BOOT`. The disk image contains everything needed for a bootable system, including:

* `/EFI/almalinux`: GRUB2 EFI and a configuration
* `/container2boot/`: kernel, initrd, rootfs—as above

The kernel cmdline arguments in `./out/esp/EFI/almalinux/grub.cfg` function as follows:

> ```txt
> root=live:PARTLABEL=C2BOOT rd.live.dir=/container2boot rd.live.image rd.live.overlay.readonly
> ```

1. The root filesystem is discovered by partition label. It is not actually the rootfs and doesn't contain `/usr` and friends, but the `live:` argument causes [`dmsquash-live`](https://github.com/dracutdevs/dracut/tree/master/modules.d/90dmsquash-live) to take over.

2. `dmsquash-live` searches for the file `/container2boot/squashfs.img`. This file is bound to a loop device and then mounted as the root filesystem.

3. Just like the previous example, a writable tmpfs layer is created to permit in-memory writes to the root filesystem.

The disk image is generated with [systemd-repart](https://www.freedesktop.org/software/systemd/man/latest/systemd-repart.service.html). Starting in Fedora 39, systemd-repart supports generating GPT disk images *without* using your kernel's loop devices. This means that unprivileged processes can generate filesystem images.

Our ESP-only scheme makes for a simple demonstration, but systemd-repart supports much more complex [configurations](https://www.freedesktop.org/software/systemd/man/latest/repart.d.html). A real deployment would probably use multiple partitions. Instead of squashfs files, `CopyBlocks` permits the generation of squashfs *partitions*.

The easiest way to boot this image is with libvirt and virt-manager.

```bash
virt-install --destroy-on-exit --transient \
  --osinfo almalinux9 --name demo \
  --ram 4096 --boot uefi --import \
  --disk out/disk.img
```

`--boot uefi` is required because the generated image does not contain MBR/legacy bootloaders. Your *host* system needs libvirt and the `edk2-ovmf` package installed to provide the UEFI firmware. By default, the UEFI in libvirt is SecureBoot Enforcing. The generated image will support SecureBoot Enforcing if its GRUB2 and kernel images are signed—and they are in AlmaLinux by default.

Normally, udisksd permits most users to mount attached USB storage devices and to obtain full read/write reign over vfat filesystems. This can permit unprivileged users to rewrite our boot media and rootfs—and we don't want this. Our [Dockerfile.machine](./Dockerfile.machine) adds an `/etc/fstab` entry to prevent this. Alternatively, you can `systemctl mask udisks2.service`.

## Bare Metal

For bare-metal boots, it is necessary to transfer the OS image to physical media. This is always a privileged operation—after all, `mount(1)` requires root and kernel access. While we can also use systemd-repart to install to bare-metal disks, this is also a privileged operation. We want to minimize the use of privileges where we can.

On desktop systems with udisksd installed and enabled, users can mount USB storage devices without privileges. The daemon performs the privileged operations on your behalf.

To make a bootable system, use GNOME Disks to partition a USB drive with at least one `FAT` filesystem. Then, click the settings button for the partition and choose "`Edit partition...`"

* Type: EFI System
* Name: `C2BOOT`

You can also format the disk from the command-line. **CAUTION**: be sure you have identified the correct `/dev` entry for your device. If you format the wrong disk, you may irreversibly **DESTROY** data that is precious to you, like your OS or home folder. In the example below, the target drive is called `/dev/sdf`. Yours will be different.

```bash
sudo gdisk /dev/sdf
```

* For a new partition,

  1. `d` to delete the partition table
  2. `n` to add a new partition
  3. Input partition sizes or hit ENTER to accept the defaults
  4. For hex code or GUID, enter `EF00` (EFI system partition)
  5. `c` to change the partition name to `C2BOOT`
  6. `w` to write the partition table

* For an existing partition,

  1. `c` to change the partition name to `C2BOOT`.
  2. `t` to change the partition type code to `EF00` (EFI system partition)
  3. `w` to write the partition table

Once your device is formatted, mount it. Nautilus won't show you EFI system partitions, but you can still mount them in either GNOME Disks or via the command-line:

```bash
udisksctl mount -b /dev/sdf1

# - OR -

sudo mount /dev/sdf1 /mnt/somewhere
```

Then, just copy the contents of `out/esp` to your device.

```bash
rsync --archive out/esp/. /mnt/somewhere
```

Unmount it and you'll be all set! Instruct your UEFI-enabled BIOS to boot from your new device.

## Closing Remarks

This tutorial demonstrates a process for converting containers into fully bootable operating systems. It is only a proof-of-concept, and there are a number of concerns that any production deployment must address:

* **Containers as starting points**: container base images are often designed with the assumption that they *won't* be made into bootable operating systems. To avoid unpleasant surprises, you should understand how your chosen base image is created "`FROM scratch`." For Fedora-based containers, review the pertinent [kickstart files](https://pagure.io/fedora-kickstarts/tree/main).

* **Updates**: the demo OS does not formalize an update process. The only way to "update" it is to create a fresh image and reboot. Updates are a *necessity*, not a luxury. If you want live updates, use a complete product like [FCOS](https://fedoraproject.org/coreos/) instead.

* **Runtime immutability**: the demo OS has a full read-write overlay over the root filesystem. If you want to write-protect any portion of the filesystem, such as `/usr`, you will need to use a different partitioning and mounting scheme.

* **Persistent storage**: additional partitions and filesystems must be added for data that must survive a reboot.

* **Filesystem performance**: squashfs is not a high-performance filesystem. If your kernel and userspace tools support it, consider if [`erofs`](https://man.archlinux.org/man/mkfs.erofs.1.en) is a better fit for your application.

These concerns are left as an exercise to the reader.

In general, you will probably be better off installing a stock operating system the "normal" way and using provisioning scripts (or Ansible) to get what you want. If you're looking to run services that are already containerized, you might want a full k8s clustering solution instead.