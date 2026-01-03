---
title: "dma-buf 简述与案例"
# weight: 2
---

## 背景
为提升不同子系统间的内存共享效率，避免多次拷贝带来的性能损耗，kernel设计一种dmabuf机制。

本文作简要说明并给出demo，实现用户态与内核态的数据共享。

## 原理简述
dmabuf 主要通过`fd`实现内存数据的共享，而非避免直接拷贝数据。即`producer`与`consumer`均可通过`fd`定位到同一块共享内存数据。

显然，`fd`的大小远少于内存数据，可显著降低性能损耗。

## 案例
编写内核模块demo，在内核空间分配内存，并导出为dmabuf。

同时，创建一个misc设备，用户空间通过ioctl获取dmabuf对应的fd，实现内存共享。
```c
#include "asm-generic/fcntl.h"
#include "linux/dma-buf-map.h"
#include "linux/err.h"
#include "linux/export.h"
#include "linux/gfp.h"
#include "linux/kern_levels.h"
#include "linux/miscdevice.h"
#include "linux/printk.h"
#include <linux/module.h>
#include <linux/dma-buf.h>
#include <linux/fs.h>
#include <linux/slab.h>
#include <linux/mutex.h>

#define DEVICE_NAME "zyp_dev"
#define MY_IOCTL_GET_DMABUF_FD _IOR('a', 1, int)

static struct dma_buf *global_dmabuf = NULL;
static void *kernel_buffer;
const char *init_data = "Data inited by kernel!";

static int demo_dma_buf_ops_attach(struct dma_buf *dmabuf, struct mda_buf_attachment *attach)
{
    pr_info("dma_buf: buffer attached\n");
    return 0;
}

static struct sg_table *demo_dma_buf_ops_map(struct dma_buf_attachment *attach, enum dma_data_direction dir)
{
    pr_info("dma_buf: buffer mapped\n");
    return NULL;
}

static void demo_dma_buf_ops_unmap(struct dma_buf_attachment *attach, struct sg_table *sgt, enum dma_data_direction dir)
{
    pr_info("dma_buf: buffer unmapped\n");
}

static int demo_dma_buf_ops_mmap(struct dma_buf *dmabuf, struct vm_area_struct *vma)
{
    int ret;

    printk(KERN_INFO "[NOTICE] dma_buf: mmap request\n");

    ret = remap_pfn_range(vma, vma->vm_start, virt_to_phys(kernel_buffer) >> PAGE_SHIFT, 
                          vma->vm_end - vma->vm_start, vma->vm_page_prot);
    if (ret) {
        printk(KERN_INFO "error in remap_pfn_range!\n");
        return -EAGAIN;
    }

    return 0;
}

static int demo_dma_buf_ops_vmap(struct dma_buf *dmabuf, struct dma_buf_map *map){
    struct demo_dma_buf *demo_buf = dmabuf->priv;
    if (!demo_buf) {
        pr_err("demo_dma_buf_vmap err\n");
        return -ENOMEM;
    }

    dma_buf_map_set_vaddr(map, demo_buf);
    pr_info("demo_dma_buf_vmap: Successfully map\n");
    return 0;
}

static void demo_dma_buf_ops_vunmap(struct dma_buf *dmabuf, struct dma_buf_map *map) {
    pr_info("dma_buf: vunmap\n");
    return;
}

static void demo_dma_buf_ops_release(struct dma_buf *dmabuf)
{
    struct demo_dma_buf *demo_buf = dmabuf->priv;

    pr_info("dma_buf: buffer released\n");

    kfree(demo_buf);
}

static const struct dma_buf_ops demo_dma_buf_ops = {
    .attach = demo_dma_buf_ops_attach,
    .map_dma_buf = demo_dma_buf_ops_map,
    .unmap_dma_buf = demo_dma_buf_ops_unmap,
    .mmap = demo_dma_buf_ops_mmap,
    .vmap = demo_dma_buf_ops_vmap,
    .vunmap = demo_dma_buf_ops_vunmap,
    .release = demo_dma_buf_ops_release,
};

static long dev_ioctl(struct file *file, unsigned int cmd, unsigned long arg){
    int fd;

    DEFINE_DMA_BUF_EXPORT_INFO(exp_info);

    switch (cmd) {
        case MY_IOCTL_GET_DMABUF_FD:
            printk(KERN_INFO "Userspace prog call ioctl\n");    

            exp_info.ops = &demo_dma_buf_ops;
            exp_info.size = 4096;
            exp_info.flags = O_RDWR;
            exp_info.priv = kernel_buffer;

            global_dmabuf = dma_buf_export(&exp_info);
            if (IS_ERR(global_dmabuf)) {
                kfree(kernel_buffer);
                return PTR_ERR(global_dmabuf);
            }

            fd = dma_buf_fd(global_dmabuf, O_RDWR);

            if (fd < 0) {
                kfree(kernel_buffer);
                return fd;
            }

            printk(KERN_INFO "fd in kernel is %d\n", fd);

            if (copy_to_user((int __user *)arg, &fd, sizeof(fd))) {
                return -EFAULT;
            }
            break;
        default:
            return -EINVAL;
    }
    return 0;
}

static ssize_t my_misc_read(struct file *file, char __user *buf, size_t len, loff_t *offset) {
    printk(KERN_INFO "in misc read: %s\n", kernel_buffer);
    return 0;
}

static const struct file_operations fops = {
    .owner = THIS_MODULE,
    .unlocked_ioctl = dev_ioctl,
    .read = my_misc_read,
};

static struct miscdevice dmabuf_device_test = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = DEVICE_NAME,
    .fops = &fops,
};

static int __init demo_dma_buf_init(void)
{
    int ret = -1;
    ret = misc_register(&dmabuf_device_test);
    if (ret < 0) {
        printk(KERN_ERR "Failed to register dev\n");
    }

    kernel_buffer = kzalloc(4096, GFP_KERNEL);
    if (!kernel_buffer){
        printk(KERN_ERR "kzalloc failed!");
        return -1;
    }
    strncpy(kernel_buffer, init_data, 4095);   

    return 0;
}

static void __exit demo_dma_buf_exit(void)
{
    misc_deregister(&dmabuf_device_test);
    if (global_dmabuf)
        dma_buf_put(global_dmabuf);
    
    pr_info("dma_buf: Exiting demo_dma_buf module\n");
}

module_init(demo_dma_buf_init);
module_exit(demo_dma_buf_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("yeping");
MODULE_DESCRIPTION("A simple dma-buf demo module with fd export");
```
编译上述模块并加载至qemu中：
```bash
# insmod zyp_dmabuf.ko
```
由于自编译的kernel不包含udev程序，需手动mknod
```bash
# cat /proc/misc
125 zyp_dev
126 cpu_dma_latency
196 vfio
200 tun
237 loop-control
235 autofs
231 snapshot
127 vga_arbiter

# mknod /dev/zyp_dev c 10 125
此时/dev/目录下出现设备zyp_dev
```
与其同时，编写一个用户空间程序，使用`ioctl`操作上述设备
```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/mman.h>

#define DEVICE_NAME "/dev/zyp_dev"
#define MY_IOCTL_GET_DMABUF_FD _IOR('a', 1, int)

int main() {
    int fd, dma_fd;
    void *user_buffer;

    // 打开设备文件
    fd = open(DEVICE_NAME, O_RDWR);
    if (fd < 0) {
        perror("Failed to open the device");
        return -1;
    }

    // 使用 ioctl 获取 DMA-BUF 文件描述符
    if (ioctl(fd, MY_IOCTL_GET_DMABUF_FD, &dma_fd) < 0) {
        perror("Failed to get DMA-BUF FD");
        close(fd);
        return -1;
    }
    printf("Received DMA-BUF FD: %d\n", dma_fd);

    // 使用 mmap 映射 DMA-BUF 到用户空间
    user_buffer = mmap(0, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, dma_fd, 0);
    if (user_buffer == MAP_FAILED) {
        printf("mmap failed");
        close(dma_fd);
        close(fd);
        return -1;
    }
    printf("DMA-BUF mapped successfully at address: %p\n", user_buffer);

    printf("DMA-BUF: %s\n", (char *)user_buffer);

    printf("Writing to DMA-BUF\n");
    snprintf((char *)user_buffer, 4096, "Hello from user space!");

    // 读取 DMA-BUF 数据
    printf("DMA-BUF: %s\n", (char *)user_buffer);
    sleep(10);

    close(dma_fd);
    close(fd);

    return 0;
}
```
在完成misc设备挂载后，执行此程序
```bash
# /user_prog
[  288.646701] Userspace prog call ioctl
[  288.647376] fd in kernel is 4
Received DMA-BUF FD: 4
[  288.649275] [NOTICE] dma_buf: mmap request
DMA-BUF mapped successfully at address: 0x7f9b223000

DMA-BUF: Data inited by kernel! // 读取到由内核初始化的原始数据

Writing to DMA-BUF
DMA-BUF: Hello from user space!  // 用户空间下修改该dmabuf数据
[  298.658187] dma_buf: buffer released
```
在用户程序执行完成后，`cat`上述设备可观察到dmabuf存储的数据被成功修改
```bash
# cat /dev/zyp_dev
[  682.808552] in misc read: Hello from user space!
```