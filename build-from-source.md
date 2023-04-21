# Build-From-Source Guide

To simplify the AE procedure and reduce the total time cost, we have provided pre-built binaries (e.g., FPGA/Linux/DuVisor/QEMU images) for AE in the `firesim_scripts` repository.

But we also provide this guide for those who would like to build from source. This guide describes how to build Rocket-Chip, Linux kernels (including host and guest) as well as userspace hypervisor (including DuVisor and QEMU) on an AWS EC2 instance with AMI ID `ami-0a431711dc2c08355`. Again, for simplicity, we also launched such a instance so that a reviewer can access it by providing her public key.

Note: it may take a long time to build from source, especially for the Rocket-Chip image (< 12 hours).

<!--ts-->
<!--te-->

## Prerequisite

### Source RISC-V and FireSim Toolchains

By default, RISC-V and FireSim toolchains required for building is not set up in the environment variables. So please execute this command before further building.

```bash
cd ~/firesim && source sourceme-f1-manager.sh && cd -
```

### Copy Pre-defined Configurations

Replace original firmsim-provided configurations with our pre-defined configurations to skip various `menuconfig`-like operations.

```bash
git clone https://github.com/IPADS-DuVisor/ae-build-configs.git

# for backup
mv ~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/base-workloads/br-base/linux-config \
	~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/base-workloads/br-base/old-linux-config

cp ./ae-build-configs/linux-config ~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/base-workloads/br-base
```

### Post-building Operations

After images are built, one should replace the corresponding pre-built images in the `firesim_scripts` on the AE client side (represented by `$FIRESIM_CLIENT` in the following sections).

## OpenSBI

Replace original firmsim-provided OpenSBI with our vPMP-supported version.

```bash
# for backup
mv ~/firesim/target-design/chipyard/software/firemarshal/boards/default/firmware/opensbi \
	~/firesim/target-design/chipyard/software/firemarshal/boards/default/firmware/old-opensbi

git clone https://github.com/IPADS-DuVisor/ae-opensbi.git opensbi
# copy to the location that firesim can find our opensbi 
mv opensbi ~/firesim/target-design/chipyard/software/firemarshal/boards/default/firmware
```

## KVM host Linux

The version of the original firmsim-provided Linux is so low that does not even support KVM. We upgraded the Linux version and added support for exit-less interrupt virtualization.

```bash
git clone https://github.com/IPADS-DuVisor/ae-linux-kvm-5.16.git linux-kvm-5.16
# use git log to check tags for microbenchmarks
cd linux-kvm-5.16 && git checkout ae-kvm-opt && cd -
# copy to the location that firesim can find our host linux 
mv linux-kvm-5.16 ~/firesim/target-design/chipyard/software/firemarshal/boards/firechip

# for backup
mv ~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/drivers/iceblk-driver \
	~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/drivers/old-iceblk-driver

# copy to the location that firesim can find our host linux 
git clone https://github.com/IPADS-DuVisor/ae-iceblk-driver.git iceblk-driver
# upgrade iceblk-driver for Linux-5.16
cd iceblk-driver && git checkout ae-kvm-5.16 && cd -
mv iceblk-driver ~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/drivers

# br-base.json configures the path of the target Linux
cp ./ae-build-configs/br-base.json.kvm \
	~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/base-workloads/br-base.json
cd ~/firesim/target-design/chipyard/software/firemarshal
./marshal build br-base.json

# copy ~/firesim/target-design/chipyard/software/firemarshal/images/br-base-bin to $FIRESIM_CLIENT/br-base-bin-kvm
```

## DuVisor host Linux

Similar to KVM host Linux, we added DV-driver and userspace network driver support in the host Linux for DuVisor.

```bash
git clone https://github.com/IPADS-DuVisor/ae-linux-laputa.git linux-laputa
mv linux-laputa ~/firesim/target-design/chipyard/software/firemarshal/boards/firechip

# skip this if the iceblk-driver is already cloned and copied
mv ~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/drivers/iceblk-driver \
	~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/drivers/old-iceblk-driver
git clone https://github.com/IPADS-DuVisor/ae-iceblk-driver.git iceblk-driver
# change iceblk-driver for DuVisor
cd iceblk-driver && git checkout ae-duvisor && cd -
mv iceblk-driver ~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/drivers

# br-base.json configures the path of the target Linux
cp ./ae-build-configs/br-base.json.dv \
	~/firesim/target-design/chipyard/software/firemarshal/boards/firechip/base-workloads/br-base.json
cd ~/firesim/target-design/chipyard/software/firemarshal
./marshal build br-base.json

# copy ~/firesim/target-design/chipyard/software/firemarshal/images/br-base-bin to $FIRESIM_CLIENT/br-base-bin-laputa
```

## Guest Linux for KVM

```bash
git clone https://github.com/IPADS-DuVisor/ae-guest-linux.git guest-linux
# use git log to check tags for microbenchmarks
cd guest-linux && git checkout ae-kvm-opt && cd -
mkdir -p build-guest
cp ae-build-configs/kvm-guest-config build-guest/.config

make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- \
	-C guest-linux O=`pwd`/build-guest -j$(shell nproc)

# copy build-guest/arch/riscv/boot/Image to $FIRESIM_CLIENT/mnt-firesim/
```

## Guest Linux for DuVisor

```bash
git clone https://github.com/IPADS-DuVisor/ae-guest-linux-laputa.git guest-duvisor
mkdir -p build-guest-duvisor
cp ae-build-configs/dv-guest-config build-guest-duvisor/.config

make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- \
	-C guest-linux O=`pwd`/build-guest-duvisor -j$(shell nproc)

# copy build-guest/arch/riscv/boot/Image to $FIRESIM_CLIENT/mnt-firesim/laputa/
```

## Guest Kernel Module

```bash
git clone https://github.com/IPADS-DuVisor/ae-kmod.git guest-kmod

make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- \
	-C $(PWD)/build-guest/ M=$(PWD)/guest-kmod/ modules

# copy build-guest/arch/riscv/boot/Image to $FIRESIM_CLIENT/firesim-scripts/scripts-rootfs/
```

## DuVisor

```bash
git clone git@github.com:IPADS-DuVisor/ae-duvisor.git duvisor
# use git log to check tags for microbenchmarks
cd duvisor && git checkout ae-app && cd -

cd duvisor
# use script for docker building
./auto-mid.sh

# copy ./mnt-firesim/laputa/laputa to $FIRESIM_CLIENT/mnt-firesim/laputa/
```

## QEMU

Use static build due to lack of dynamic libraries in the virtual disk image.

```bash
git clone git@github.com:IPADS-DuVisor/ae-qemu.git qemu

cd qemu
./configure --target-list=riscv64-softmmu --enable-kvm --static \
	--disable-linux-io-uring --disable-libiscsi --disable-glusterfs \
	--disable-libusb --disable-usb-redir --audio-drv-list= --disable-opengl

make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j$(shell nproc) 

# copy ./build/qemu-system-riscv64 to $FIRESIM_CLIENT/mnt-firesim/
```

## RocketChip

### Prepare environment 
```bash
# setup env vars
cd ~/firesim
source source sourceme-f1-manager.sh 
# checkout branch to hardware with full Duvisor hardware features
cd target-design/chipyard/generators/rocket-chip
git checkout vipi-vssip-firesim
```

### Specify hardware name for firesim build
```
cd ~/firesim
vim deploy/config_build.ini
# change Duvisor under entry [builds] to your own name, for example Duvisor-ipads. Results should be similar to:
#
# [builds]
# Duvisor-ipads
vim deploy/config_build_recipes.ini
# add new entry with name you specified in deploy/config_build.ini. An example is:
#
# [Duvisor-ipads]
# DESIGN=FireSim
# TARGET_CONFIG=WithNIC_DDR3FRFCFSLLC4MB_WithDefaultFireSimBridges_WithFireSimHighPerfConfigTweaks_chipyard.QuadRocketConfig
# PLATFORM_CONFIG=WithAutoILA_F90MHz_BaseF1Config
# instancetype=z1d.2xlarge
# deploytriplet=None
```

### Run build scripts
```bash
cd ~/firesim
firesim buildafi
```

### Get agfi from build log
You can get agfi ID from the build log after the hardware build process completes.
The agfi ID is similar to the following:
```
[Duvisor-ipads]
agfi=agfi-XXXXXXXXXXXXXXXXXX
deploytripletoverride=None
customruntimeconfig=None
```

Now You can use this agfi ID as the Duvisor Hardware to run Duvisor evaluation.

### Build hardware with no-pmp feature
Repeat Step 1 "Prepare environment", but change branch to no PMP. The scripts are as follows:
```bash
# setup env vars
cd ~/firesim
source source sourceme-f1-manager.sh 
# checkout branch to hardware with Duvisor hardware without PMP features
cd target-design/chipyard/generators/rocket-chip
git checkout nopmp
```
