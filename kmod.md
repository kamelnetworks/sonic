# Kernel modules


## Pre-req

You need the following packages to build:

```
dpkg-dev dh-python stgit python3-jinja2 kernel-wedge quilt flex bison cpio quilt libelf-dev libssl-dev bc
```

## Build kernel

Download the `sonic-linux-kernel` repository and check out the branch/commit you
want to build. Build using `make`, some platforms require options but defaults
seem fine for a normal amd64 switch.

## Build platform modules

In this example we will build the modules `platform/marvell-teralynx/sonic-platform-modules-cel/silverstone-x/modules`.

```
make -C [..]/sonic-linux-kernel/linux-*/debian/build/build_amd64_none_amd64 \
  M=[..]/sonic-buildimage/platform/marvell-teralynx/sonic-platform-modules-cel/silverstone-x/modules
```
