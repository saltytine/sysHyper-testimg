# Test environment description
Image comes from the Linux kernel compiled by yjy and cxy, and the kernel version is 5.4.0.
The matching jailhouse.ko and config files have been patched with RVM1.5,
rootfs comes from buildroot.

## Start process
The qemu startup command is
```sh
qemu-system-aarch64 \
    -drive file=./rootfs.qcow2,discard=unmap,if=none,id=disk,format=qcow2 \
    -device virtio-blk-device,drive=disk \
    -m 1G -serial mon:stdio \
    -kernelImage\
    -append "root=/dev/vda mem=768M" \
    -cpu cortex-a57 -smp 4 -nographic -machine virt,gic-version=3,virtualization=on \
    -device virtio-serial-device -device virtconsole,chardev=con \
    -chardev vc,id=con \
    -netnic\
    -net user,hostfwd=tcp::2333-:22
```

After qemu is started, transfer the files in the guest to the guest linux
```sh
scp -P 2333 -r test-img/guest/* root@localhost:~/
```

Among them, the gic-demo.bin file is the image used by the second cell

Transfer the hypervisor image to the guest
```sh
make scp
```

## Compilation instructions for each component
Please refer to the document [here](https://github.com/saltytine/notes-and-guides/blob/main/arm64-qemu-jailhouse.md)
The following also gives a relatively simple compilation process

### Cross-compilation tool
To compile arm64 programs on x86 machines, you need to use a cross-compilation tool, here we use gcc-aarch64-linux-gnu
```sh
sudo apt-cache search aarch64 # View the installable versions
sudo apt-get install gcc-aarch64-linux-gnu # Install a default version
```

### kernel compilation
Take 5.4.0 as an example

```sh
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.tar.xz
tar -xvJf linux-5.4.tar.xz
cd linux-5.4
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

### rootfs compilation
Use the buildroot tool, which can generate a complete Linux system, but we only use it to generate the root file system here.

>Buildroot can automatically build a complete and bootable Linux for embedded systems. Buildroot is mainly used for small or embedded systems. Buildroot can automatically build the required cross-compilation tool chain, create a root file system, compile a Linux kernel image, and generate a boot loader for the target embedded system.
```sh
wget https://buildroot.org/downloads/buildroot-2023.02.3.tar.xz
tar -xvJf buildroot-2023.02.3.tar.xz
cd buildroot-2023.02.3
make menuconfig
```

The following settings need to be made in the menuconfig interface:
```
Target Architecture (AArch64 (little endian)) Target Architecture Variant (cortex-A57) (root) Root password (ttyAMA0) TTY port
[*] dhcp (ISC)
[*] dhcp client
[*] dhcpcd
[*] dropbear
[*] ext2/3/4 root filesystem ext2/3/4 variant (ext4) (1G) exact size
```

After the configuration is completed, just make it directly. The generated files are located in output/images
```
sudo make
cd output/images
```

The rootfs.ext4 is the generated file system, with a size of 1G, which is the same as the previous `(1G) exact size`. This determines the initial capacity of the file system and can be customized. The actual size occupied is less than 1G. In order to save space, we can convert it to qcow2 format.
```sh
qemu-img convert -O qcow2 rootfs.ext4 rootfs.qcow2
```

## jailhouse compilation
jailhouse can also be cross-compiled, but you need to have a compiled kernel project first. In addition, some modifications have been made to jailhouse in this project, so you need to apply a patch first. The specific commands are as follows:
```sh
git clone https://github.com/siemens/jailhouse.git
cd jailhouse
git checkout v0.10
patch -f -p1 < path/to/jailhouse.patch
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- KDIR=path/to/kernel -j$(nproc)
```

After the compilation is completed, you can see the corresponding generated files in each folder, among which:
- The xxx.cell file in configs/arm64 is the vm configuration file, which is used when enabling and creating cells
- driver/jailhouse.ko is loaded into root The kernel module of the cell provides some management functions, provides an interface for the user state through ioctl operations, and uses hypercall to call the interface provided by the hypervisor.
- hypervisor/jailhouse.bin is the hypervisor image of jailhouse, running in EL2, providing the core functions of virtualization. The sysHyper of this project is mainly to replace it.
- tools/jailhouse is the user state program of the root cell, which is used to call the kernel module jailhouse.ko and is a user-oriented management interface.
- inmates contains some runnable demos
