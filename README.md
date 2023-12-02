# container2boot: Simple and Easy Bootable Containers

This project is a proof-of-concept which generates a bootable LiveCD from an ordinary, non-bootable AlmaLinux container. This project is heavily inspired by the most excellent [livecd-tools](https://github.com/livecd-tools/livecd-tools) and [d2vm](https://github.com/linka-cloud/d2vm). Unlike the existing approaches, `container2boot` aims to use 100% unprivileged containers and require only:

* `podman >= 4.3` with buildah
* access to public container registries and the AlmaLinux DNF repositories
