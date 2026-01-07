---
title: "自制 UEFI Bootloader 并启动 ARM64 Linux Kernel"
---

## 概述

UEFI, The Unified Extensible Firmware Interface, 定义 OS 与 硬件 之间的接口标准。
UEFI 在 OS 与 platform firmware 之间扮演中间层：

- OS 可调用 UEFI 接口获取信息，完成 boot 等操作
- platform firmware 做 UEFI 接口的具体实现

UEFI 规范允许 OS 直接运行于多种平台（需支持对应的CPU架构），无需额外适配 firmware 或 OS.

> UEFI 是接口标准，不包含具体实现
> The specification defines the set of interfaces and structures that the platform firmware must implement.

遵循 UEFI 规范可编写一系列 UEFI App。其中，Bootloader 是一种核心 UEFI App，支持启动相应的 OS。
> 若通过固件直接启动OS，固件需兼容各种OS、文件系统、内核格式等。
> 且升级OS也需要升级固件，依赖过强
> Bootloader 再次起到 中间层 作用


为方便理解 UEFI，本文首先编写一个简单的 UEFI App 并运行，展示 UEFI 开发基本流程。
随后，编写简单的 Bootloader 程序（也是一种UEFI App），加载并运行 Linux Kernel.

## 基础环境搭建

最终目标：在 x86 Ubuntu QEMU 上 运行 UEFI App，并加载 Arm64 Linux Kernel.

1. clang 编译工具链、QEMU、busybox 静态链接编译、Linux 内核编译
	参考：https://wonderzyp.github.io/misc/qemu_usb/
2. 安装 QEMU UEFI firmware
```bash
sudo apt install qemu-efi-aarch64

# QEMU UEFI 固件程序
cp /usr/share/AAVMF/AAVMF_CODE.fd .

# UEFI 变量区，后续操作用不上，可不复制
cp /usr/share/AAVMF/AAVMF_VARS.fd . 
```
> AAVMF, Arm Architecture Virtual Machine Firmware

## UEFI Application 编写

源码：
```c
// in uefi_app.c

struct EFI_SYSTEM_TABLE {
    char _buf[60]; // 占位符，手动保证后续成员 *ConOut 偏移量正确
    struct EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL {
        unsigned long long _buf;
        unsigned long long (*OutputString)(
            struct EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL *This,
            unsigned short *String);
        unsigned long long _buf2[4];
        unsigned long long (*ClearScreen)(
             struct EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL *This);
    } *ConOut;
};
  
void efi_main(void *ImageHandle __attribute__ ((unused)),
    struct EFI_SYSTEM_TABLE *SystemTable)
{
    SystemTable->ConOut->ClearScreen(SystemTable->ConOut);
    SystemTable->ConOut->OutputString(SystemTable->ConOut, u"Hello UEFI!\r\n");
    while (1);
}

```
编译：
```bash
clang --target=aarch64-pc-windows-msvc -ffreestanding -fshort-wchar -nostdlib -c uefi_app.c -o uefi_app.o

lld-link /subsystem:efi_application /entry:efi_main /out:BOOTAA64.EFI uefi_app.o
```

将生成 `BOOTAA64.EFI` 文件，查看格式：
```bash
> file BOOTAA64.EFI
BOOTAA64.EFI: PE32+ executable (EFI application) Aarch64, for MS Windows
```

>UEFI uses a subset of PE32+ image format with a modified header signature.

在当前目录新建 `esp/EFI/BOOT/` 目录，并将 `BOOTAA64.EFI` 移动至 BOOT 目录下：
```
 ~/workplace/uefi_c  tree .
.
├── AAVMF_CODE.fd
├── esp
│   ├── EFI
│   │   └── BOOT
│   │       ├── BOOTAA64.EFI
```

启动 QEMU：
```bash
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -m 1024 \
  -nographic \
  -drive if=pflash,format=raw,readonly=on,file=AAVMF_CODE.fd \
  -drive format=raw,file=fat:rw:esp \
  -net none
```

一段时间后，屏幕输出 `Hello UEFI!` 字符串。

### UEFI 接口与实现的关联

考虑一个问题：上述源码及编译链接过程，并没有看到 `OutputString` 等接口具体实现或lib库文件，那么这个UEFI App是如何通过编译并运行的？

**函数指针** 是解决上述问题的关键：
```c
void efi_main(void *ImageHandle __attribute__ ((unused)),
    struct EFI_SYSTEM_TABLE *SystemTable) {
	// ...
	SystemTable->ConOut->ClearScreen(SystemTable->ConOut);
}
```
此处 `ClearScreen`并不是具体的函数调用，而是一个函数指针：
```c
// 结构体内定义如下：
unsigned long long (*OutputString)(
	struct EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL *This,
	unsigned short *String);
```
对应到汇编代码，类似取一个地址并跳转：
```asm
ldr x0, [SystemTable + offset]   // 取函数指针
blr x0                           // 间接跳转
```
显然，不涉及任何外部符号名，因此无需指定任何lib文件。

进一步的，如果是跳转地址，则结构体内的偏移量不容有误。
分析结构体定义：
```c
struct EFI_SYSTEM_TABLE {
    char _buf[60]; // 占位符，手动保证后续成员 *ConOut 偏移量正确
    struct EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL {
        unsigned long long _buf;
        unsigned long long (*OutputString)(
            struct EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL *This,
            unsigned short *String);
        unsigned long long _buf2[4];
        unsigned long long (*ClearScreen)(
             struct EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL *This);
    } *ConOut;
};
```
可以看到有两个占位成员：`_buf[60]` 与 `_buf2[4]`。
旨在保证此处使用到的 结构体成员，在结构体内的偏移量正确。

分析第一个 `char _buf[60]`的占位原理：
UEFI 标准定义了 `struct EFI_SYSTEM_TABLE`的所有内容：
```c
typedef struct {
  EFI_TABLE_HEADER                   Hdr; // 24 bytes
  CHAR16                             *FirmwareVendor;  // 8 bytes
  UINT32                             FirmwareRevision; // 4 bytes
  EFI_HANDLE                         ConsoleInHandle;  // 8 bytes
  EFI_SIMPLE_TEXT_INPUT_PROTOCOL     *ConIn; // 8 bytes
  EFI_HANDLE                         ConsoleOutHandle; // 8 bytes


  EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL    *ConOut;
  
  EFI_HANDLE                         StandardErrorHandle;
  // ...
  EFI_CONFIGURATION_TABLE            *ConfigurationTable;
} EFI_SYSTEM_TABLE;
```

此处仅使用 `EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL *ConOut`成员，只需保证结构体内相对偏移量正确。
前面共有 6 个尚未使用的结构体成员，占用 60 bytes，与声明的占位符 `char _buf[60]`匹配。
> 实际工程开发不建议手动计算并写死占位符

### 自动加载 BOOTAA64.EFI 原理

UEFI 规范规定，缺少其他启动项时，加载默认路径下的文件：
`\EFI\BOOT\{file_name}`
各架构下 `{file_name}` 值如下：

| Architecture   | File name        |
| -------------- | ---------------- |
| Intel 32-bit   | BOOTIA32.EFI     |
| X86_64         | BOOTX64.EFI      |
| Itanium        | BOOTIA64.EFI     |
| AArch32        | BOOTARM.EFI      |
| AArch64        | BOOTAA64.EFI     |
| RISC-V 32-bit  | BOOTRISCV32.EFI  |
| RISC-V 64-bit  | BOOTRISCV64.EFI  |
| RISC-V 128-bit | BOOTRISCV128.EFI |

## 简易 Bootloader 编写

实现一个简易 Bootloader，仅实现两个功能：
1. 定位并加载 Linux Kernel Image
2. 传入 内核启动参数 并 启动 Image

项目整体源码：
https://github.com/wonderzyp/demo_for_blog/tree/main/uefi_c

重点分析 `main.c` 中加载镜像的逻辑：
```c
void efi_main(void *ImageHandle, struct EFI_SYSTEM_TABLE *SystemTable)
{
    struct EFI_LOADED_IMAGE_PROTOCOL *lip;
    struct EFI_LOADED_IMAGE_PROTOCOL *lip_aarch64_image;
    struct EFI_DEVICE_PATH_PROTOCOL *dev_path;
    struct EFI_DEVICE_PATH_PROTOCOL *dev_node;
	struct EFI_DEVICE_PATH_PROTOCOL *dev_path_merged;
    unsigned long long status;
    void *image;
    unsigned short options[] = L"root=/dev/vdb rw init=/init";

    efi_init(SystemTable);
    ST->ConOut->ClearScreen(ST->ConOut);
	
	// 初始化相关 Protocol
    status = ST->BootServices->OpenProtocol(
        ImageHandle, &lip_guid, (void **)&lip, ImageHandle, NULL, EFI_OPEN_PROTOCOL_GET_PROTOCOL);
    assert(status, L"OpenProtocol(lip)");

    status = ST->BootServices->OpenProtocol(
        lip->DeviceHandle, &dpp_guid, (void **)&dev_path, ImageHandle,
        NULL, EFI_OPEN_PROTOCOL_GET_PROTOCOL);
    assert(status, L"OpenProtocol(dpp)");

    dev_node = DPFTP->ConvertTextToDeviceNode(L"Image");
    dev_path_merged = DPUP->AppendDeviceNode(dev_path, dev_node);
	
	// 确定内核镜像的设备路径
	puts(L"dev_path_merged: ");
	puts(DPTTP->ConvertDevicePathToText(dev_path_merged, FALSE, FALSE));
	puts(L"\r\n");

	// 加载镜像
    status = ST->BootServices->LoadImage(FALSE, ImageHandle, dev_path_merged, NULL, 0, &image);
    assert(status, L"LoadImage");
    puts(L"LoadImage: Success!\r\n");

    status = ST->BootServices->OpenProtocol(
        image, &lip_guid, (void **)&lip_aarch64_image, ImageHandle, NULL, EFI_OPEN_PROTOCOL_BY_HANDLE_PROTOCOL);
    assert(status, L"OpenProtocol(lip_image)");
    
    // 传入启动参数并运行内核
    lip_aarch64_image->LoadOptions = options;
    lip_aarch64_image->LoadOptionsSize = (strlen(options) + 1) * sizeof(unsigned short);

    status = ST->BootServices->StartImage(image, NULL, NULL);
    assert(status, L"StartImage");
    puts(L"StartImage: Success!\r\n");

	while (TRUE);
}
```

涉及两个核心概念：Protocols 与 Device Paths.
### Protocols
Protocols 提供一系列操作 handles 的功能，handles 对应 资源 (a disk drive, USB device and so on...)

**Protocols**: interfaces that provide **functions** to interact with a resource.
**Handles**: An opaque pointer represent resource. To operate on a handle you have to open a protocol.

### Device Paths
设备路径 与 常见的文件系统路径 不同，不能直接通过 文件系统路径 加载 Images.

移动 `esp/EFI/BOOT/` 目录下的 `BOOTAA64.EFI`至 `esp/`下，启动QEMU将不会自动加载。
此时进入 UEFI Shell:
```bash
UEFI Interactive Shell v2.2
EDK II
UEFI v2.70 (EDK II, 0x00010000)
Mapping table
      FS0: Alias(s):HD0b:;BLK1:
          PciRoot(0x0)/Pci(0x1,0x0)/HD(1,MBR,0xBE1AFDFA,0x3F,0xFBFC1)
     BLK2: Alias(s):
          VenHw(93E34C7E-B50E-11DF-9223-2443DFD72085,00)
     BLK0: Alias(s):
          PciRoot(0x0)/Pci(0x1,0x0)
Press ESC in 3 seconds to skip startup.nsh or any other key to continue.
Shell>
```

上述的 `PciRoot(0x0)/Pci(0x1,0x0)/HD(1,MBR,0xBE1AFDFA,0x3F,0xFBFC1)` 即为一个设备：
```bash
Shell> fs0:
FS0:\> ls
Directory of: FS0:\
01/07/2026  11:23               4,608  BOOTAA64.EFI
FS0:\> 026  15:20 <DIR>         8,192  EFI
          1 File(s)       4,608 bytes
          1 Dir(s

# 可直接运行此 EFI 文件
FS0:\> BOOTAA64.EFI
```

显然，定位文件 `BOOTAA64.EFI` 需同时指定 设备(`fs0`) + 路径（根目录）。

> 设备信息解读：
> PciRoot(0x0)/Pci(0x1,0x0)/HD(1,MBR,0xBE1AFDFA,0x3F,0xFBFC1)
> 
> PCI Root 0 下 的 PCI设备(Bus 1, Device 0)上，有一块磁盘。
> 其MBR分区表的第一个分区。
> 分区起始 LBA = `0x3F`，长度 `0xFBFC1`，分区签名 `0xBE1AFDFA`


## 联通 Bootloader 与 Linux Kernel

### 编译 Linux Kernel
编译前需开启几个关键 Kconfig：
```
CONFIG_VIRTIO=y
CONFIG_VIRTIO_BLK=y
CONFIG_VIRTIO_MMIO=y

CONFIG_EFI=y
CONFIG_EFI_STUB=y
CONFIG_EFIVAR_FS=y

CONFIG_EXT4_FS=y
CONFIG_EXT4_FS_POSIX_ACL=y
```

将编译好的镜像，拷贝至esp目录：
```
cp <kernel_source_path>/arch/arm64/boot/Image ./esp/
```

### 制作简易 rootfs
```bash
# current PWD: /home/zyp/workplace/uefi_c/

dd if=/dev/zero of=rootfs.ext4 bs=1M count=64

mkfs.ext4 rootfs.ext4

mkdir mnt
sudo mount rootfs.ext4 ./mnt
sudo mkdir -p mnt/{bin,sbin,etc,proc,sys,dev,tmp}

sudo tee mnt/init << 'EOF'
#!/bin/sh
echo "Booted into minimal rootfs"
exec /bin/sh
EOF

sudo chmod 777 init
```

需自行编译一个 **static** 的 busybox 文件：
```bash
sudo cp <path-to-busybox> ./mnt/bin/
cd mnt/bin/
sudo ln -s busybox sh

sudo umount mnt
```
至此，rootfs 已制作完成，启动 QEMU：
```bash
qemu-system-aarch64 \
    -machine virt \
    -cpu cortex-a57 \
    -m 1024 \
    -nographic \
    -drive if=pflash,format=raw,readonly=on,file=AAVMF_CODE.fd \
    -drive format=raw,file=fat:rw:esp \
    -drive file=rootfs.ext4,format=raw,if=virtio \
    -net none
```

顺利启动日志：
```
dev_path_merged: PciRoot(0x0)/Pci(0x1,0x0)/HD(1,MBR,0xBE1AFDFA,0x3F,0xFBFC1)/Image
LoadImage: Success!
EFI stub: Booting Linux Kernel...
EFI stub: EFI_RNG_PROTOCOL unavailable
EFI stub: Generating empty DTB
EFI stub: Exiting boot services...

[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x411fd070]
[    0.000000] Linux version 5.15.179 (yep@hp) (Ubuntu clang version 14.0.0-1ubuntu1.1, Ubuntu LLD 14.0.0) #7 SMP PREEMPT Tue Jan 6 16:15:07 CST 2026
[    0.000000] efi: EFI v2.70 by EDK II
[    0.000000] efi: SMBIOS 3.0=0x7bed0000 MEMATTR=0x792ee698 ACPI 2.0=0x78430018 MEMRESERVE=0x7842ff18
[    0.000000] ACPI: Early table checksum verification disabled
[    0.000000] ACPI: RSDP 0x0000000078430018 000024 (v02 BOCHS )

// ...

[    0.593525] VFS: Mounted root (ext4 filesystem) on device 254:16.
[    0.595354] devtmpfs: mounted
[    0.627192] Freeing unused kernel memory: 1600K
[    0.627949] Run /init as init process
Booted into minimal rootfs
/bin/sh: can't access tty; job control turned off
~ #
~ # busybox ls
bin         etc         lost+found  sbin        tmp
dev         init        proc        sys
```


参考：

1. https://kagurazakakotori.github.io/ubmp-cn/
2. https://rust-osdev.github.io/uefi-rs/
