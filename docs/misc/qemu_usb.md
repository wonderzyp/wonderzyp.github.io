---
title: "基于QEMU的USB kernel调试环境搭建"
weight: 2
---

## 背景

为方便调试Kernel内USB的相关模块，本文介绍基于QEMU调试USB设备的方式。

实现在QEMU内运行ARM64 kernel，识别U盘并进行读写等操作。

## 基本环境

- Ubuntu 20
- QEMU emulator version 8.2.2
- clang version 18.1.8
- buildroot-2024.05.2
- Kernel Source Code: linux-5.15.163


## 步骤
主要包括：内核源码编译、内核模块编译、rootfs编译与QEMU启动四部分。

### 内核源码编译

源码: https://kernel.org/
下载完成后依次执行：
```c
make menuconfig ARCH=arm64 LLVM=1
// 建议开启以下Kconfig
// CONFIG_USB_ANNOUNCE_NEW_DEVICES
// CONFIG_USB_UAS
// CONFIG_USB_STORAGE
// CONFIG_STACKTRACE

make ARCH=arm64 LLVM=1 -j16
```

> 注：Clang等编译环境需提前配置好，这里不赘述


### 内核模块编译
在调试的过程中，发现识别U盘依赖部分内核模块。

默认的内核编译命令会编译使能的模块，但生成的ko文件分散在各个文件夹内。可在编译时传入参数INSTALL_MOD_PATH，指定内核模块的存储位置，方便后续存放至rootfs内

```bash
make ARCH=arm64 LLVM=1 INSTALL_MOD_PATH=/home/zyp/workplace/linux-5.15.163 modules_install -j16
```

执行后，编译生成的模块将被存放至kernel源码根目录的`lib/modules/`下

为加快运行速度，可仅保留与调试USB相关的内核模块，即`kernel/drivers/usb/`文件夹

```bash
zyp@hp ~/workplace/linux-5.15.163/lib/modules/5.15.163-g7c2cad57e872$ tree -L 4
.
├── build -> /home/zyp/workplace/linux-5.15.163
├── kernel
│   └── drivers
│       └── usb
│           ├── class
│           ├── gadget
│           ├── host
│           ├── renesas_usbhs
│           ├── serial
│           └── typec
├── modules.alias  
├── modules.alias.bin  
├── modules.builtin  
├── modules.builtin.alias.bin  
├── modules.builtin.bin  
├── modules.builtin.modinfo  
├── modules.dep  
├── modules.dep.bin  
├── modules.devname  
├── modules.order  
├── modules.softdep  
├── modules.symbols  
├── modules.symbols.bin  
└── source -> /home/zyp/workplace/linux-5.15.163
```

### rootfs 准备

本文采用buildroot配置rootfs: https://buildroot.org/

相较于busybox，buildroot提供更全面的内置功能支持

```bash
zyp@hp ~/workplace/buildroot-2024.05.2$ make menuconfig
// 几个关键配置项如下
Target options  --->
    Target Architecture (AArch64 (little endian))
    Target Architecture Variant (cortex-A72)

Toolchain  --->
    Kernel Headers (Linux 5.15.x kernel headers) // 按需配置
    [*] Enable C++ support
    [*] Enable compiler OpenMP support

System configuration  --->
    (qemu-kernel) System hostname // 随意
    (Welcome to QEMU Kernel) System banner

Kernel  --->
    [ ] Linux Kernel // 勿选中，kernel自行编译


// 配置完成后可直接执行make
zyp@hp ~/workplace/buildroot-2024.05.2$ make
```

编译时间较长，完成后产物为`output/images/rootfs.tar`

在根目录创建 rootfs 文件夹，用于存放解压 rootfs.tar 的内容

```bash
mkdir ~/rootfs
cd rootfs
tar -xvf rootfs.tar
rm rootfs.tar

zyp@hp ~/rootfs$ ls
bin  dev  etc  lib  lib64  linuxrc  media  mnt  opt  proc  root  run  sbin  sys  tmp  usr  var
```

修改部分文件配置：
```bash
// 同目录下复制一个init文件，用作启动时的init进程（这里暂不清楚为何在qemu启动参数中，直接指定linuxrc不可行）
zyp@hp ~/rootfs$ cp linuxrc init

// 修改/etc/fstab文件，结尾新增一项用于动态识别/dev/sdaX
devtmpfs    /dev        devtmpfs defaults 0 0
```

### QEMU配置与启动
以某U盘为例，查询其相关信息，用于传入QEMU启动参数

```bash
# 在Host环境查询其vendorID与productID
zyp@hp ~/workplace/qemu_kernel_wp$ lsusb
Bus 004 Device 003: ID 0781:55a9 SanDisk Corp.  SanDisk 3.2Gen1
```

调试USB时，需在QEMU启动时执行部分USB相关配置参数，以保证USB的正常功能。

以下为参考脚本，实际使用时需自行修改相关文件路径

```bash
#!/bin/bash

#set -x 
function start_qemu {
    /home/zyp/software/qemu-8.2.2/build/qemu-system-aarch64 \
                    -M virt \
                    -cpu cortex-a72 \
                    -smp 4 \
                    -m 4G \
                    -kernel ./Image \
                    -initrd ./initrd-busybox.img \
                    -device qemu-xhci \
                    -device usb-host,vendorid=0x0781,productid=0x55a9 \
                    -nographic \
                    -append "init=/init nokaslr console=ttyAMA0"
}


function prepare_Image {
    cp /home/zyp/workplace/linux-5.15.163/arch/arm64/boot/Image .
    echo "Copy Kernel Image Successfully!"
}

function prepare_VFS {
    orig_dir=$(pwd)
    target_dir="/home/zyp/rootfs"
    pushd "$target_dir"

    find . -print0 | cpio --null -ov --format=newc | /home/zyp/software/pigz/pigz -9 > ../initrd-busybox.img
    cp ../initrd-busybox.img $orig_dir

    popd

    echo "Prepare VFS Successfully!"
}

function prepare_ko {
    rootfs_dir="/home/zyp/rootfs/"
    kernel_dir="/home/zyp/workplace/linux-5.15.163/"
    find $kernel_dir -name "zyp*ko" -exec cp {} $rootfs_dir \;
    echo "Copy Kernel Modules Successfully!"
}

function prepare_modules {
    rootfs_lib_dir="/home/zyp/rootfs/lib/"
    modules_dir="/home/zyp/workplace/linux-5.15.163/lib/modules"
    cp -r $modules_dir $rootfs_lib_dir
    echo "Copy ko for modprobe successfully!"
}

function init_env {
    prepare_Image
    prepare_ko
    prepare_modules
    prepare_VFS
}


init_env
#set +x
start_qemu
```

启动后有类似下述输出：

```bash
Welcome to QEMU Kernel
qemu-kernel login: root
#
```
欲识别USB设备，需先加载相关内核模块(`xhci-pci`)
```bash
# modprobe xhci-pci
[   50.580266] xhci_hcd 0000:00:02.0: xHCI Host Controller
[   50.580852] xhci_hcd 0000:00:02.0: new USB bus registered, assigned bus number 1
[   50.585094] xhci_hcd 0000:00:02.0: hcc params 0x00087001 hci version 0x100 quirks 0x0000000000000010
[   50.587794] xhci_hcd 0000:00:02.0: xHCI Host Controller
[   50.587934] xhci_hcd 0000:00:02.0: new USB bus registered, assigned bus number 2
[   50.588136] xhci_hcd 0000:00:02.0: Host supports USB 3.0 SuperSpeed
[   50.591668] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 5.15
[   50.591859] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[   50.592005] usb usb1: Product: xHCI Host Controller
[   50.592116] usb usb1: Manufacturer: Linux 5.15.163-g7c2cad57e872 xhci-hcd
[   50.592252] usb usb1: SerialNumber: 0000:00:02.0
[   50.595835] hub 1-0:1.0: USB hub found
[   50.596448] hub 1-0:1.0: 4 ports detected
[   50.600566] usb usb2: We don't know the algorithms for LPM for this host, disabling LPM.
[   50.601038] usb usb2: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 5.15
[   50.601203] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[   50.601338] usb usb2: Product: xHCI Host Controller
[   50.601428] usb usb2: Manufacturer: Linux 5.15.163-g7c2cad57e872 xhci-hcd
[   50.601547] usb usb2: SerialNumber: 0000:00:02.0
[   50.602501] hub 2-0:1.0: USB hub found
[   50.602778] hub 2-0:1.0: 4 ports detected
# [   50.944031] usb 2-1: new SuperSpeed USB device number 2 using xhci_hcd
[   50.968823] usb 2-1: New USB device found, idVendor=0781, idProduct=55a9, bcdDevice= 1.00
[   50.969059] usb 2-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[   50.969205] usb 2-1: Product:  SanDisk 3.2Gen1
[   50.969301] usb 2-1: Manufacturer:  USB
[   50.969403] usb 2-1: SerialNumber: 0401ecfc7dff80cacbb13dfeef8d232f74924cbcc749be0b88ca65e183b126dd9c5000000000000000000000a0f8d9f5ff0e2b18a955810761aca95d
[   50.972923] usb-storage 2-1:1.0: USB Mass Storage device detected
[   50.975211] scsi host0: usb-storage 2-1:1.0
[   52.013784] scsi 0:0:0:0: Direct-Access      USB      SanDisk 3.2Gen1 1.00 PQ: 0 ANSI: 6
[   52.018127] sd 0:0:0:0: [sda] 120164352 512-byte logical blocks: (61.5 GB/57.3 GiB)
[   52.022135] sd 0:0:0:0: [sda] Write Protect is off
[   52.023948] sd 0:0:0:0: [sda] Write cache: disabled, read cache: enabled, doesn't support DPO or FUA
[   52.049430]  sda: sda1
[   52.061278] sd 0:0:0:0: [sda] Attached SCSI removable disk
```

此时已识别到sda设备，将其挂载至特定目录下，即可对USB进行读写等操作。

```bash
// 挂载设备
# cd /mnt
# mkdir zyp  // 用于挂载
# mount /dev/sda1 /mnt/zyp/  
[   44.609076] FAT-fs (sda1): Volume was not properly unmounted. Some data may be corrupt. Please run fsck.
```

至此，操作/mnt目录可进行相关调试工作。