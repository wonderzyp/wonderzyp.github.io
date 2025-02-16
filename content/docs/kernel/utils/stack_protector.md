---
title: "内核栈保护机制与栈溢出攻击"
weight: 2
---

## 背景
近期遇到stack protector相关panic，现调查该机制。
```Shell
Kernel panic - not syncing: stack-protector: Kernel stack is corrupted in: __schedule+0x88c/0xb8c
CPU: 0 PID: 1922 Comm: xxx Tainted: G OE #1
Call trace:
    dump_backtrace.cfi_jt+0x0/0x8
    dump_stack_lvl+0x80/0xb8
    panic+0x190/0x44c
    rcu_eqs_exit+0x0/0x90
    __schedule+0x88c/0xb8c
    0xffffffc03480ff54
```
本文将给出一个使用栈溢出修改函数执行流程的案例，并简要介绍stack-protector的基本原理。

## Hack: 通过栈溢出修改程序的执行流
ARM64 ABI约定的函数帧大致布局如下：
```Shell
High Address
   +------------------+:q
   | Caller x30 (LR)  |
   +------------------+
   | Caller x29 (FP)  |
   +------------------+
   | Local Vars       |
   | ...              |
   +------------------+
Low Address
```
显然，只要能获取FP的地址，即可通过FP+8获取LR地址。
修改LR地址内容，指向特定的函数，即可实现程序控制流变动。

> 此外，局部变量地址与FP, LR紧邻，理论上可通过局部变量地址获取LR地址。
> 局部变量若出现溢出，可能会影响函数帧的LR等内容，造成问题。

下述为内核demo示例，为模拟栈溢出方式，通过局部变量buffer的地址，确定并修改LR数值：

```C
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/delay.h>

#define BUF_SIZE 8

void hacked_function(void) {
    ssleep(1);
    printk(KERN_INFO "Hacked! LR is changed manully via local params!\n");
}

void vulnerable_function(void) {
    char buffer[BUF_SIZE];
    printk(KERN_INFO "Buffer addr: %px\n", buffer);

    printk(KERN_INFO "Current func fp: %px\n", (void *)__builtin_frame_address(0));
    printk(KERN_INFO "Origin func lr: %px\n", (char *)__builtin_frame_address(0) + 8);

    // 借助局部变量buffer地址，定位LR并修改内容
    void** ret_addr  = (void **)(buffer + 24);
    *ret_addr = hacked_function;

    printk(KERN_INFO "The vulnerable function has been completed!\n");
}

static int __init stack_overflow_init(void) {
    vulnerable_function();
    return 0;
}

static void __exit stack_overflow_exit(void) {
    printk(KERN_INFO "Unloading stack overflow proc module...\n");
}

module_init(stack_overflow_init);
module_exit(stack_overflow_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("yep");
MODULE_DESCRIPTION("Kernel Stack Overflow DEMO");
MODULE_VERSION("1.0");
```

编译并加载该内核模块，将输出以下信息：

```Shell
# insmod yep_stack_corrupt.ko
[    5.377790] Buffer addr: ffffffc00a27ba80
[    5.378054] Current func fp: ffffffc00a27ba90
[    5.378168] Origin func lr: ffffffc00a27ba98
[    5.378267] The vulnerable function has been completed!
[    6.412074] Hacked! LR is changed manully via local params!
[    7.436055] Hacked! LR is changed manully via local params!
[    8.460046] Hacked! LR is changed manully via local params!
[    9.483760] Hacked! LR is changed manully via local params!
```

至此，成功修改函数的执行流程。

## stack-protector机制简述
从上述案例可知，若某个局部变量未妥善处理导致溢出，可能会覆盖函数栈的LR等寄存器数据，造成严重的问题。

为缓解这种情况，stack-protector在保存局部变量与寄存器的栈地址之间，存储一个特定数值canary。返回前将检测该数值是否变动（栈溢出的情况， 在影响到寄存器区域前， 大概率先覆盖canary值）

### 原理

反汇编上述ko文件，关键代码如下：

```Assembly
0000000000000038 <vulnerable_function>:
  38:   d503245f        bti     c
  3c:   d503201f        nop
  40:   d503201f        nop
  44:   d503233f        paciasp
  48:   d10083ff        sub     sp, sp, #0x20
  4c:   a9017bfd        stp     x29, x30, [sp, #16]
  50:   910043fd        add     x29, sp, #0x10
  54:   d5384108        mrs     x8, sp_el0
  58:   90000000        adrp    x0, 0 <hacked_function>
  5c:   91000000        add     x0, x0, #0x0
  60:   f9424108        ldr     x8, [x8, #1152]
  64:   910003e1        mov     x1, sp
  68:   f90007e8        str     x8, [sp, #8]
//.....
  b4:   d5384108        mrs     x8, sp_el0
  b8:   f9424108        ldr     x8, [x8, #1152]
  bc:   f94007e9        ldr     x9, [sp, #8]
  c0:   eb09011f        cmp     x8, x9
  c4:   540000a1        b.ne    d8 <vulnerable_function+0xa0>  // b.any
//...
  d4:   d65f03c0        ret
  d8:   94000000        bl      0 <__stack_chk_fail>
```
由上可知，x8将特定值存储在`[sp, #8]`处
执行完成后，将判断`[sp, #8]`处存储的数值是否有变动，若发生变动，将跳转`__stack_chk_fail`，触发stack-protector机制。

### 手动触发

基于上述案例，局部变量buffer+8处的值，模拟栈溢出行为。

```Diff
diff --git a/drivers/yep_stack_corrupt/yep_stack_corrupt.c b/drivers/yep_stack_corrupt/yep_stack_corrupt.c
index f3cbbb82a..d9c559aea 100644
--- a/drivers/yep_stack_corrupt/yep_stack_corrupt.c
+++ b/drivers/yep_stack_corrupt/yep_stack_corrupt.c
@@ -22,6 +22,8 @@ void vulnerable_function(void) {
     void** ret_addr  = (void **)(buffer + 24);
     *ret_addr = hacked_function;
 
+    *(buffer+8) = 'z'; // buffer溢出，触发stack-protector机制
+
     printk(KERN_INFO "The vulnerable function has been completed!\n");
 }
```

由于会影响到`[sp, #8]`的数值，应触发panic:

```Bash
# insmod yep_stack_corrupt.ko
[    7.922556] Buffer addr: ffffffc00a58ba80
[    7.922985] Current func fp: ffffffc00a58ba90
[    7.923158] Origin func lr: ffffffc00a58ba98
[    7.923323] The vulnerable function has been completed!
[    7.924263] Kernel panic - not syncing: stack-protector: Kernel stack is corrupted in: vulnerable_function+0xa4/0xa4 [yep_stack_corrupt]
[    7.925069] CPU: 0 PID: 161 Comm: insmod Not tainted 5.15.176-gc23cb937769c-dirty #14
[    7.925475] Hardware name: linux,dummy-virt (DT)
[    7.925923] Call trace:
[    7.926061]  dump_backtrace+0x0/0x1f4
[    7.926302]  show_stack+0x24/0x30
[    7.926442]  dump_stack_lvl+0x64/0x7c
[    7.926598]  dump_stack+0x18/0x38
[    7.926730]  panic+0x15c/0x3a8
[    7.926864]  rcu_dynticks_inc+0x0/0x58
[    7.927346]  cleanup_module+0x0/0xed0 [yep_stack_corrupt]
[    7.927569]  hacked_function+0x0/0x38 [yep_stack_corrupt]
[    7.927789]  do_one_initcall+0xd0/0x2e4
```

> 补充：此demo中buffer + 24为LR地址, +16对应FP
> buffer自身长度为8, 对应地址buffer[0..7]
> buffer+8即处于寄存器与局部变量之间