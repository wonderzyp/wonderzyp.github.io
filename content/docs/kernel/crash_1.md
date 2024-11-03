---
title: "案例-基于crash-utility分析非法地址访问panic"
weight: 1
---

## 背景
工作中遇到一例因访问非法地址导致的内核panic，本文记录使用crash工具调查问题的整体过程。

panic的内核log如下:
```text
[90251.511294] Unable to handle kernel paging request at virtual address 0030343220706448
[90251.511330] Mem abort info:
[90251.511392]   ESR = 0x96000004
[90251.511540]   EC = 0x25: DABT (current EL), IL = 32 bits
[90251.511622]   SET = 0, FnV = 0
[90251.511691]   EA = 0, S1PTW = 0
[90251.511727]   FSC = 0x04: level 0 translation fault
[90251.511748] Data abort info:
[90251.511800]   ISV = 0, ISS = 0x00000004
[90251.511832]   CM = 0, WnR = 0
[90251.511882] [0030343220706448] address between user and kernel address ranges
[90251.511928] Internal error: Oops: 0000000096000004 [#1] PREEMPT SMP
[90251.512184] CPU: 4 PID: 1981 Comm: InputDispatcher Tainted: G        W  OE
[90251.512248] pstate: 804000c5 (Nzcv daIF +PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[90251.512328] pc : enqueue_hrtimer+0x6c/0x1b4
[90251.512365] lr : __run_hrtimer+0x19c/0x4a0
[90251.512491] sp : ffffffc00af0bdf0
[90251.512522] x29: ffffffc00af0be00 x28: ffffff80e728b600 x27: ffffff89989cb400
[90251.512554] x26: 00005215505cc8a0 x25: ffffff80e728b600 x24: ffffffc0097f6e80
[90251.512582] x23: 00000000000000c0 x22: 0000000000000001 x21: ffffff89989cb3c0
[90251.512619] x20: ffffff89989cb998 x19: ffffff89989cb400 x18: ffffffc00aef5038
[90251.512665] x17: 0000000000000000 x16: 0000000000000000 x15: 0000000000000001
[90251.512706] x14: 0000000000000010 x13: 0000000000000010 x12: 6430343220706430
[90251.512738] x11: 0000521550991300 x10: 0000000000000008 x9 : 0000000000000001
[90251.512762] x8 : ffffff89989cb420 x7 : 00000b4000000000 x6 : ffffff81f93081b0
[90251.512793] x5 : ffffffc0128b3770 x4 : 0000000000000001 x3 : 000000000000000a
[90251.512824] x2 : 0000000000000000 x1 : ffffff89989cb400 x0 : ffffff89989cb998
[90251.512855] Call trace:
[90251.512884]  enqueue_hrtimer+0x6c/0x1b4
[90251.512908]  __run_hrtimer+0x19c/0x4a0
[90251.512935]  hrtimer_interrupt+0x2fc/0x580
[90251.512978]  arch_timer_handler_virt+0x5c/0xa0
[90251.513010]  handle_percpu_devid_irq+0xc0/0x374
[90251.513049]  handle_domain_irq+0x9c/0x120
[90251.513071]  gic_handle_irq.35512+0x58/0x26c
[90251.513103]  call_on_irq_stack+0x3c/0x70
[90251.513151]  do_interrupt_handler+0x44/0xa0
[90251.513181]  el1_interrupt+0x34/0x64
[90251.513206]  el1h_64_irq_handler+0x1c/0x2c
[90251.513234]  el1h_64_irq+0x7c/0x80
[90251.513326]  __wake_up+0xdc/0x1c0
[90251.513496]  binder_wakeup_thread_ilocked+0x64/0x158
[90251.513578]  binder_proc_transaction+0x428/0xc5c
[90251.513651]  binder_transaction+0x1d8c/0x29d8
[90251.513704]  binder_thread_write+0xa30/0x23f4
[90251.513776]  binder_ioctl_write_read+0xac/0x588
[90251.513826]  binder_ioctl+0x1b0/0xee8
[90251.513862]  __arm64_sys_ioctl+0x184/0x20c
[90251.513907]  invoke_syscall+0x60/0x150
[90251.513943]  el0_svc_common+0x8c/0xf8
[90251.513978]  do_el0_svc+0x28/0x98
[90251.514007]  el0_svc+0x24/0x84
[90251.514039]  el0t_64_sync_handler+0x88/0xec
[90251.514080]  el0t_64_sync+0x1b8/0x1bc
[90251.514127] Code: 5280010a 5280020d aa0b03ec f9400c0b (f9400d8e)
[90251.514177] ---[ end trace 02c260193d0c1796 ]---
[90251.514233] Kernel panic - not syncing: Oops: Fatal exception in interrupt
```

## 调查记录
追踪最后一条指令，寻找异常地址`0030343220706448`的来源
```text
[90251.512328] pc : enqueue_hrtimer+0x6c/0x1b4
[90251.512365] lr : __run_hrtimer+0x19c/0x4a0
[90251.512491] sp : ffffffc00af0bdf0
[90251.512522] x29: ffffffc00af0be00 x28: ffffff80e728b600 x27: ffffff89989cb400
[90251.512554] x26: 00005215505cc8a0 x25: ffffff80e728b600 x24: ffffffc0097f6e80
[90251.512582] x23: 00000000000000c0 x22: 0000000000000001 x21: ffffff89989cb3c0
[90251.512619] x20: ffffff89989cb998 x19: ffffff89989cb400 x18: ffffffc00aef5038
[90251.512665] x17: 0000000000000000 x16: 0000000000000000 x15: 0000000000000001
[90251.512706] x14: 0000000000000010 x13: 0000000000000010 x12: 6430343220706430
[90251.512738] x11: 0000521550991300 x10: 0000000000000008 x9 : 0000000000000001
[90251.512762] x8 : ffffff89989cb420 x7 : 00000b4000000000 x6 : ffffff81f93081b0
[90251.512793] x5 : ffffffc0128b3770 x4 : 0000000000000001 x3 : 000000000000000a
[90251.512824] x2 : 0000000000000000 x1 : ffffff89989cb400 x0 : ffffff89989cb998
[90251.512855] Call trace:
[90251.512884]  enqueue_hrtimer+0x6c/0x1b4
[90251.512908]  __run_hrtimer+0x19c/0x4a0

crash> sym enqueue_hrtimer
ffffffc00826832c (t) enqueue_hrtimer out/msm-kernel-autogvm-gki/gki_kernel/common/kernel/time/hrtimer.c: 1083

// ffffffc00826832c + 0x6c = 0xffffffc008268398
// 0x6c = 108
```
获取`enqueue_hrtimer`的汇编代码，可知panic前执行的汇编语句为`ldr x14, [x12, #24]`：

```text
crash> dis ffffffc00826832c
0xffffffc00826832c <enqueue_hrtimer>:   paciasp
0xffffffc008268330 <enqueue_hrtimer+4>: sub     sp, sp, #0x40
0xffffffc008268334 <enqueue_hrtimer+8>: str     x30, [x18], #8
0xffffffc008268338 <enqueue_hrtimer+12>:        stp     x29, x30, [sp, #16]
0xffffffc00826833c <enqueue_hrtimer+16>:        add     x29, sp, #0x10
0xffffffc008268340 <enqueue_hrtimer+20>:        stp     x22, x21, [sp, #32]
0xffffffc008268344 <enqueue_hrtimer+24>:        stp     x20, x19, [sp, #48]
0xffffffc008268348 <enqueue_hrtimer+28>:        nop
0xffffffc00826834c <enqueue_hrtimer+32>:        ldr     x8, [x1]
0xffffffc008268350 <enqueue_hrtimer+36>:        mov     w11, #0x1                       // #1
0xffffffc008268354 <enqueue_hrtimer+40>:        ldr     w9, [x1, #8]
0xffffffc008268358 <enqueue_hrtimer+44>:        ldr     w10, [x8, #8]
0xffffffc00826835c <enqueue_hrtimer+48>:        lsl     w9, w11, w9
0xffffffc008268360 <enqueue_hrtimer+52>:        orr     w9, w10, w9
0xffffffc008268364 <enqueue_hrtimer+56>:        str     w9, [x8, #8]
0xffffffc008268368 <enqueue_hrtimer+60>:        add     x8, x1, #0x20
0xffffffc00826836c <enqueue_hrtimer+64>:        strb    w11, [x0, #56]
0xffffffc008268370 <enqueue_hrtimer+68>:        ldr     x9, [x0]
0xffffffc008268374 <enqueue_hrtimer+72>:        cmp     x9, x0
0xffffffc008268378 <enqueue_hrtimer+76>:        b.ne    0xffffffc0082683cc <enqueue_hrtimer+160>  // b.any
0xffffffc00826837c <enqueue_hrtimer+80>:        ldr     x11, [x8]
0xffffffc008268380 <enqueue_hrtimer+84>:        cbz     x11, 0xffffffc0082683d8 <enqueue_hrtimer+172>
0xffffffc008268384 <enqueue_hrtimer+88>:        mov     w9, #0x1                        // #1
0xffffffc008268388 <enqueue_hrtimer+92>:        mov     w10, #0x8                       // #8
0xffffffc00826838c <enqueue_hrtimer+96>:        mov     w13, #0x10                      // #16
0xffffffc008268390 <enqueue_hrtimer+100>:       mov     x12, x11
0xffffffc008268394 <enqueue_hrtimer+104>:       ldr     x11, [x0, #24]
0xffffffc008268398 <enqueue_hrtimer+108>:       ldr     x14, [x12, #24]
0xffffffc00826839c <enqueue_hrtimer+112>:       cmp     x11, x14
// ...
```
该指令含义：将`x12`寄存器的值加上24，得到新的地址。读取该地址存储的数据并放至`x14`内。 显然，此处`x12`+24得到的值为非法地址，进而导致读取产生panic，证明如下：
```text
// 先判断x12来源
0xffffffc00826837c <enqueue_hrtimer+80>:        ldr     x11, [x8]
0xffffffc008268380 <enqueue_hrtimer+84>:        cbz     x11, 0xffffffc0082683d8 <enqueue_hrtimer+172>
//...
0xffffffc008268390 <enqueue_hrtimer+100>:       mov     x12, x11
0xffffffc008268394 <enqueue_hrtimer+104>:       ldr     x11, [x0, #24]
0xffffffc008268398 <enqueue_hrtimer+108>:       ldr     x14, [x12, #24]
```

`x12`寄存器的值源于`x11`，而`x11`的值源于`x8`

从崩溃调用栈看，`x8`在崩溃前值并未发生改变，可直接读取以还原`x11`的值:


```text
[90251.512762] x8 : ffffff89989cb420 x7 : 00000b4000000000 x6 : ffffff81f93081b0
[90251.512793] x5 : ffffffc0128b3770 x4 : 0000000000000001 x3 : 000000000000000a
[90251.512824] x2 : 0000000000000000 x1 : ffffff89989cb400 x0 : ffffff89989cb998


crash> rd ffffff89989cb420
ffffff89989cb420:  ffffffc0242cbb78                    x.,$....
```

即在执行`mov x12, x11`指令时，`x11`寄存器的应为**ffffffc0242cbb78**

若按正常流程，`x12`将获取`x11`寄存器值，再读取`x12`+24地址存储数据，并传至`x14`：

```text
// x12 = x11 = ffffffc0242cbb78
// x12 + 24 = ffffffc0242cbb78 + 0x18 = ffffffc0242cbb90

crash> rd ffffffc0242cbb90
ffffffc0242cbb90:  000052157c399b2a                    *.9|.R..
```

显然，正常情况下该指令应正常执行，读取**ffffffc0242cbb90**内容，得到**000052157c399b2a**，并存入`x14`内。

这里对应的源码如下：

```text
crash> dis -s 0xffffffc008268398
FILE: lib/timerqueue.c
LINE: 22

dis: 0xffffffc008268398: source code is not available


// in file: timerqueue.c
#define __node_2_tq(_n) \
        rb_entry((_n), struct timerqueue_node, node)

static inline bool __timerqueue_less(struct rb_node *a, const struct rb_node *b)
{
        return __node_2_tq(a)->expires < __node_2_tq(b)->expires;
}
```
此处与汇编指令可对应上：

```text
0xffffffc008268394 <enqueue_hrtimer+104>:       ldr     x11, [x0, #24]
0xffffffc008268398 <enqueue_hrtimer+108>:       ldr     x14, [x12, #24]
0xffffffc00826839c <enqueue_hrtimer+112>:       cmp     x11, x14

crash> struct -o timerqueue_node
struct timerqueue_node {
   [0] struct rb_node node;
  [24] ktime_t expires;
}
SIZE: 32
```
若按描述的正常流程看，`x0`与`x12`应为`timerqueue_node`地址，`x11`与`x14`基于此+24偏移量，为`expires`成员

但问题现场`x12`寄存器为**6430343220706430**，最终导致访问非法地址**0x6430343220706448**
> (0x6430343220706430+0x18 = 6430343220706448)

与panic报错信息地址相符：
> Unable to handle kernel paging request at virtual address 0030343220706448

### 直接原因：寄存器x12数据异常，导致访问非法地址
从已有guestdump难以进一步调查，初步判断为***Use After Free***类型问题，借助KASAN版本排查

使能KASAN后，相同测试场景必定触发KASAN机制检测，摘取关键log如下：
```text
[ 1456.533420] ==================================================================
[ 1456.533540] BUG: KASAN: use-after-free in rb_erase+0x50/0x614
[ 1456.533586] Read of size 8 at addr ffffff886f315090
[ 1456.533702] CPU: 1 PID: 2115 Tainted: G        W  OE
[ 1456.534029] Call trace:
[ 1456.534072]  dump_backtrace+0x0/0x308
[ 1456.534119]  show_stack+0x18/0x24
[ 1456.534161]  dump_stack_lvl+0x88/0xa4
[ 1456.534208]  print_address_description+0x74/0x384
[ 1456.534249]  kasan_report+0x180/0x1f0
[ 1456.534283]  __asan_load8+0xb4/0xb8
[ 1456.534337]  rb_erase+0x50/0x614  // 访问异常地址的指令
[ 1456.534378]  timerqueue_del+0x8c/0xcc
[ 1456.534416]  remove_hrtimer+0xb8/0x230
[ 1456.534454]  hrtimer_try_to_cancel+0x138/0x180
[ 1456.534493]  schedule_hrtimeout_range_clock+0x134/0x1d0
[ 1456.534544]  schedule_hrtimeout_range+0x14/0x20
[ 1456.534572]  do_select+0xd54/0xe2c
[ 1456.534701]  core_sys_select+0x3f0/0x5b0
[ 1456.534915]  __arm64_sys_pselect6+0x31c/0x33c
[ 1456.534977]  invoke_syscall+0x68/0x1b0
[ 1456.535019]  el0_svc_common+0x100/0x148
[ 1456.535057]  do_el0_svc+0x38/0xb4
[ 1456.535106]  el0_svc+0x20/0x50
[ 1456.535182]  el0t_64_sync_handler+0x68/0xac
[ 1456.535257]  el0t_64_sync+0x164/0x168
[ 1456.535300] 
[ 1456.535394] Allocated by task 18503:
[ 1456.535418]  ____kasan_kmalloc+0xd8/0x110
[ 1456.535432]  __kasan_kmalloc+0x10/0x1c
[ 1456.535443]  kmem_cache_alloc_trace+0x274/0x39c
[ 1456.535455]  common_init+0x38/0x14c [zyp]
[ 1456.535674]  __driver_probe_device+0x120/0x1fc
[ 1456.535687]  driver_probe_device+0x78/0x26c
[ 1456.535697]  __driver_attach+0x2a0/0x338
[ 1456.535706]  bus_for_each_dev+0x11c/0x19c
[ 1456.535717]  driver_attach+0x38/0x48
[ 1456.535729]  bus_add_driver+0x1d8/0x378
[ 1456.535740]  driver_register+0x18c/0x244
[ 1456.535750]  i2c_register_driver+0x74/0xf4
[ 1456.535762]  0xffffffc0026430ac
[ 1456.535773]  do_one_initcall+0x170/0x58c
[ 1456.535783]  do_init_module+0xd0/0x528
[ 1456.535795]  load_module+0x1e44/0x1f64
[ 1456.535805]  __arm64_sys_finit_module+0x1b4/0x1d0
[ 1456.535816]  invoke_syscall+0x68/0x1b0
[ 1456.535828]  el0_svc_common+0x100/0x148
[ 1456.535839]  do_el0_svc+0x38/0xb4
[ 1456.535850]  el0_svc+0x20/0x50
[ 1456.535860]  el0t_64_sync_handler+0x68/0xac
[ 1456.535871]  el0t_64_sync+0x164/0x168
[ 1456.535881] 
[ 1456.535941] Freed by task 25694:
[ 1456.535972]  kasan_set_track+0x4c/0x7c
[ 1456.535984]  kasan_set_free_info+0x28/0x4c
[ 1456.535994]  ____kasan_slab_free+0x104/0x150
[ 1456.536006]  __kasan_slab_free+0x18/0x28
[ 1456.536017]  slab_free_freelist_hook+0xe4/0x210
[ 1456.536029]  kfree+0xf0/0x2ec
[ 1456.536038]  ou1_common_deinit+0x34/0x4c [zyp]
[ 1456.536226]  i2c_device_remove+0x44/0x118
[ 1456.536238]  device_release_driver_internal+0x2c4/0x464
[ 1456.536248]  driver_detach+0x144/0x1b4
[ 1456.536257]  bus_remove_driver+0xd0/0x12c
[ 1456.536289]  cleanup_module+0x20/0x344 [zyp]
[ 1456.536383]  __arm64_sys_delete_module+0x28c/0x2e4
[ 1456.536394]  invoke_syscall+0x68/0x1b0
[ 1456.536405]  el0_svc_common+0x100/0x148
[ 1456.536416]  do_el0_svc+0x38/0xb4
[ 1456.536428]  el0_svc+0x20/0x50
[ 1456.536438]  el0t_64_sync_handler+0x68/0xac
[ 1456.536449]  el0t_64_sync+0x164/0x168
```
报错处为`hrtimer`管理`timer`实例的部分，拟将某个`timer`从`timerqueue`中移除

显然，此处的`timer`实例内存地址已被释放，若再次访问将触发KASAN的UAF机制检测

至此，已定位问题原因，需检查对应内核模块的代码流程，此处不做展开

## One More Thing
目前还存疑：对于使用已经free的内存，为什么不开启KASAN不能必现问题

对此，在本地qemu环境编写内核模块，复现问题现场
```c
// A kernel module to test Use After Free
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/hrtimer.h>
#include <linux/ktime.h>
#include <linux/init.h>
#include <linux/slab.h>
#include <linux/vmalloc.h>

#define TIMER_INTERVAL_MS 15000 // 定时器间隔，单位毫秒

struct zyp_struct {
    struct hrtimer my_timer; // 1. 声明自定义hrtimer
    char data;
};

struct zyp_struct *my_struct;

static ktime_t ktime;

enum hrtimer_restart __nocfi timer_callback(struct hrtimer *timer) {
    pr_info("hrtimer expired\n");    
    hrtimer_forward_now(timer, ktime);
    return HRTIMER_RESTART;
}

static int __init my_module_init(void) {
    my_struct = kmalloc(sizeof(struct zyp_struct), GFP_KERNEL);
    ktime = ktime_set(0, (unsigned long)TIMER_INTERVAL_MS * 1000000);
    hrtimer_init(&my_struct->my_timer, CLOCK_REALTIME, HRTIMER_MODE_REL); // 2. 初始化定时器
    my_struct->my_timer.function = timer_callback;  // 3. 绑定回调函数

    hrtimer_start(&my_struct->my_timer, ktime, HRTIMER_MODE_REL); // 4. 启动定时器
    pr_info("hrtimer loaded\n");

    return 0;
}

static void __exit my_module_exit(void) {
    kfree(my_struct);
    hrtimer_cancel(&my_struct->my_timer); // 5. 取消定时器
    pr_info("hrtimer unloaded\n");
}

module_init(my_module_init);
module_exit(my_module_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("hrtimer demo");
MODULE_AUTHOR("yeping");
```
编译该文件，并重复加载、卸载模块，发现并不会触发内核panic
```text
# insmod zyp_hrtimer.ko
[    5.845908] hrtimer loaded
# rmmod zyp_hrtimer.ko
[    8.460515] hrtimer unloaded
# insmod zyp_hrtimer.ko
[    9.724364] hrtimer loaded
# rmmod zyp_hrtimer.ko
[   10.495389] hrtimer unloaded
# insmod zyp_hrtimer.ko
[   11.240446] hrtimer loaded
# rmmod zyp_hrtimer.ko
[   11.893076] hrtimer unloaded
# insmod zyp_hrtimer.ko
[   12.668077] hrtimer loaded
# rmmod zyp_hrtimer.ko
[   13.272061] hrtimer unloaded
# insmod zyp_hrtimer.ko
[   13.980115] hrtimer loaded
# rmmod zyp_hrtimer.ko
[   14.645785] hrtimer unloaded
# insmod zyp_hrtimer.ko
[   15.313301] hrtimer loaded
# rmmod zyp_hrtimer.ko
[   15.885197] hrtimer unloaded
# insmod zyp_hrtimer.ko
[   16.536938] hrtimer loaded
# rmmod zyp_hrtimer.ko
[   17.079767] hrtimer unloaded
```

猜测即使调用`kfree`释放内存，但对应地址的数据还维持原始值，因此`hrtimer_cancel`依旧可正常调用。

基于此，在调用`hrtimer_cancel`前，使用`vmalloc`申请大块内存空间，希望将对应数据覆盖：

```diff
--- zyp_hrtimer.c       2024-10-11 17:29:10.318357243 +0800
+++ zyp_hrtimer1.c      2024-10-11 17:28:56.094187117 +0800
@@ -37,11 +37,6 @@

 static void __exit my_module_exit(void) {
     kfree(my_struct);
-    void *buffer = vmalloc(1 * 512 * 1024 * 1024);  // 500 MB
-    if (!buffer) {
-        printk(KERN_ERR "Failed to allocate 500MB of memory with vmalloc\n");
-    }
-    memset(buffer, 0x88, 1 * 512 * 1024 * 1024);
     hrtimer_cancel(&my_struct->my_timer); // 5. 取消定时器
     pr_info("hrtimer unloaded\n");
 }
```

此时再重复加载、卸载模块，发现第一次可正常卸载，第二次触发非法地址访问的panic，与预期相符
> 不是每次vmalloc均能覆盖到对应值

```text
Welcome to QEMU Kernel
qemu-kernel login: root
# insmod zyp_hrtimer.ko
[    6.436036] hrtimer loaded
# rmmod zyp_hrtimer.ko
[    8.665333] hrtimer unloaded

# insmod zyp_hrtimer.ko
[   14.203281] hrtimer loaded
# rmmod zyp_hrtimer.ko
[   20.382410] Unable to handle kernel NULL pointer dereference at virtual address 0000000000000008
[   20.382739] Mem abort info:
[   20.382829]   ESR = 0x0000000096000005
[   20.382948]   EC = 0x25: DABT (current EL), IL = 32 bits
[   20.383090]   SET = 0, FnV = 0
[   20.383193]   EA = 0, S1PTW = 0
[   20.383285]   FSC = 0x05: level 1 translation fault
[   20.383418] Data abort info:
[   20.383505]   ISV = 0, ISS = 0x00000005
[   20.383613]   CM = 0, WnR = 0
[   20.383758] user pgtable: 4k pages, 39-bit VAs, pgdp=00000001220c3000
[   20.383927] [0000000000000008] pgd=0000000000000000, p4d=0000000000000000, pud=0000000000000000
[   20.384285] Internal error: Oops: 0000000096000005 [#1] PREEMPT SMP
[   20.384575] Modules linked in: zyp_hrtimer(-) [last unloaded: zyp_hrtimer]
[   20.385176] CPU: 2 PID: 161 Comm: rmmod Not tainted 5.15.163-ga1becabc74b9-dirty #9
[   20.385432] Hardware name: linux,dummy-virt (DT)
[   20.385683] pstate: 200000c5 (nzCv daIF -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[   20.385891] pc : rb_insert_color+0x48/0x130
[   20.386142] lr : timerqueue_add+0x9c/0xbc
[   20.386284] sp : ffffffc00a25be60
[   20.386385] x29: ffffffc00a25be60 x28: 0000000000000001 x27: 00000004be07a440
[   20.386618] x26: 0000000000000001 x25: 0000000000000001 x24: ffffff80ff7bda10
[   20.386813] x23: ffffff80ff7bd4c0 x22: 0000000000000000 x21: ffffff80ff7bd500
[   20.387011] x20: ffffff80ff7bd4c0 x19: 0000000000000000 x18: ffffffc00a251038
[   20.387197] x17: 0000000000000004 x16: 0000000000000000 x15: 0000000000000008
[   20.387395] x14: 0000000000000010 x13: ffffff80e2079a00 x12: 0000000000000008
[   20.387582] x11: 0000000000000000 x10: 00000004be440d00 x9 : ffffff80e2079a00
[   20.387781] x8 : 0000000000000000 x7 : 00001e8480000000 x6 : 0000000000000000
[   20.387970] x5 : ffffffc00a1f6460 x4 : ffffffc00a1f65b0 x3 : 01b166d484b297ce
[   20.388159] x2 : 00000000003d0900 x1 : ffffff80ff7bd520 x0 : ffffff80ff7bda10
[   20.388476] Call trace:
[   20.388721]  rb_insert_color+0x48/0x130
[   20.388836]  __hrtimer_run_queues+0x160/0x1f8
[   20.388971]  hrtimer_interrupt+0xcc/0x38c
[   20.389082]  arch_timer_handler_virt+0x5c/0xa0
[   20.389212]  handle_percpu_devid_irq+0xb8/0x1f0
[   20.389336]  handle_domain_irq+0x7c/0xf0
[   20.389448]  gic_handle_irq+0x74/0x10c
[   20.389559]  call_on_irq_stack+0x3c/0x70
[   20.389685]  do_interrupt_handler+0x40/0xa0
[   20.389805]  el1_interrupt+0x38/0x68
[   20.389914]  el1h_64_irq_handler+0x20/0x30
[   20.390035]  el1h_64_irq+0x78/0x7c
[   20.390138]  get_page_from_freelist+0xa94/0xe74
[   20.390267]  __alloc_pages+0x14c/0x278
[   20.390376]  alloc_pages+0x16c/0x238
[   20.390482]  __vmalloc_node_range+0x1f0/0x38c
[   20.390603]  vmalloc+0x88/0x9c
[   20.390696]  cleanup_module+0x2c/0xe18 [zyp_hrtimer]
[   20.391125]  __arm64_sys_delete_module+0x1e0/0x274
[   20.391267]  invoke_syscall+0x64/0x154
[   20.391378]  el0_svc_common.llvm.2325351408555753798+0xb8/0xf8
[   20.391634]  do_el0_svc+0x2c/0x9c
[   20.391747]  el0_svc+0x28/0x5c
[   20.391837]  el0t_64_sync_handler+0xd4/0xf0
[   20.391968]  el0t_64_sync+0x1a8/0x1ac
[   20.392240] Code: f9000109 54fffea0 f9400128 370005a8 (f940050a)
[   20.393104] ---[ end trace 72f1d697c7efd5c0 ]---
[   20.393412] Kernel panic - not syncing: Oops: Fatal exception in interrupt
[   20.393734] SMP: stopping secondary CPUs
[   20.394257] Kernel Offset: disabled
[   20.394371] CPU features: 0x4,000800f1,00000846
[   20.394622] Memory Limit: none
[   20.394868] ---[ end Kernel panic - not syncing: Oops: Fatal exception in interrupt ]---
```
>  一般的，使用`kfree`后会将对应指针置空。是否存在一种可能：在`kfree`函数与置空操作的中途，被调度或中断，进而导致奇怪问题。