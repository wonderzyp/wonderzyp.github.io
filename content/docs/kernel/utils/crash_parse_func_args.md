---
title: "crash-utility动态调试: 函数参数解析"
weight: 2
---

## 背景
系统发生崩溃时，获取调用栈中函数的实际参数值有助于分析问题，具有较高应用价值。

实际的函数参数可能处于以下位置：
- CPU寄存器
- 函数调用栈

本文以某次panic的现场为例，分别演示从寄存器与函数栈帧中，解析某个函数参数值的流程。

## 解析位于寄存器的变量
通过crash的log可知，该例是由于RT throttling而触发crash

```Bash
sched: RT throttling activated for rt_rq ffffff8983caf480 (cpu 1)
rt_period_timer: expires=756598519060 now=756560049761 runtime=950000000 period=1000000000
potential CPU hogs:
current droid.bluetooth (15585) is running for 1005648284 nsec
droid.bluetooth (15585)
```

bt命令获取崩溃前上下文：

```Plain
crash> bt
PID: 15585    TASK: ffffff87fa61d980  CPU: 1    COMMAND: "droid.bluetooth"
 #0 [ffffffc01000b580] __delay at ffffffc0113da944
 #1 [ffffffc01000b5c0] __const_udelay at ffffffc0113daa18
 #2 [ffffffc01000b5e0] qcom_wdt_trigger_bite at ffffffc010940594
 #3 [ffffffc01000b600] do_vm_restart at ffffffc0101bba80
 #4 [ffffffc01000b620] atomic_notifier_call_chain at ffffffc01031f2c0
 #5 [ffffffc01000b660] machine_restart at ffffffc0102bac94
 #6 [ffffffc01000b780] panic at ffffffc0102e77ac
 #7 [ffffffc01000b830] die at ffffffc0102c4850
 #8 [ffffffc01000b860] bug_handler$387e226de7fcca202d1582dafa94315e at ffffffc0102c57c8
 #9 [ffffffc01000b890] brk_handler$04d00b2f5508655b50c86c3318e430f9 at ffffffc0102b7ec4
#10 [ffffffc01000b8b0] do_debug_exception at ffffffc0100a8974
#11 [ffffffc01000ba30] el1_dbg at ffffffc0100aa508
     PC: ffffffc01034a55c  [dump_throttled_rt_tasks+504]
     LR: ffffffc01034a55c  [dump_throttled_rt_tasks+504]
     SP: ffffffc01000ba40  PSTATE: 60400085
    X29: ffffffc01000bc40  X28: ffffff8983ca1bc0  X27: ffffffc012499240
    X26: ffffff8983caf240  X25: ffffff8983caf490  X24: ffffffc01000bc40
    X23: ffffff8983caf490  X22: ffffffc011d5485b  X21: ffffffc01000bb55
    X20: 0000000000000064  X19: ffffff8983caf480  X18: ffffffc01000d060
    X17: ffffff8227fe7e80  X16: 0000000000000280  X15: da2f3fd121cbcfbe
    X14: 0000000000000000  X13: 0000000000000000  X12: 0000000000000000
    X11: 00000000000000e0  X10: 0000000000000000   X9: e919127ca995d800
     X8: e919127ca995d800   X7: 0000000000000000   X6: ffffffc0103876dc
     X5: 0000000000000000   X4: 0000000000000080   X3: 0000000000000000
     X2: ffffff8983c9b318   X1: 0000000000000000   X0: 0000000000000109
#12 [ffffffc01000bc40] dump_throttled_rt_tasks at ffffffc01034a558
#13 [ffffffc01000bca0] update_curr_rt$c8e64db50a637c5c2b38997a086ae425 at ffffffc0103486b0
#14 [ffffffc01000bce0] task_tick_rt$c8e64db50a637c5c2b38997a086ae425 at ffffffc010347e98
#15 [ffffffc01000bd30] scheduler_tick at ffffffc01032ec98
#16 [ffffffc01000bd90] update_process_times at ffffffc0103b2fec
#17 [ffffffc01000bdc0] tick_sched_timer$2e93e54c57d54c141bd5e65a4951d56c at ffffffc0103cb148
#18 [ffffffc01000be30] __hrtimer_run_queues at ffffffc0103b5f3c
#19 [ffffffc01000beb0] hrtimer_interrupt at ffffffc0103b5b38
#20 [ffffffc01000bf20] arch_timer_handler_virt$bcb11a1e6d2eb4313d99baca138ae8af at ffffffc010d5fbd8
#21 [ffffffc01000bf40] handle_percpu_devid_irq at ffffffc0103932ac
#22 [ffffffc01000bf90] __handle_domain_irq at ffffffc01038aaa0
#23 [ffffffc01000bfd0] gic_handle_irq$7f58c51dd0f0d487d89dbb027abc57c5 at ffffffc0100a8c14
```

可知，`dump_throttled_rt_tasks`为crash前最后一次函数调用

分析相应源码：
```C

static void dump_throttled_rt_tasks(struct rt_rq *rt_rq)
{
    struct rt_prio_array *array = &rt_rq->active;
    struct sched_rt_entity *rt_se;
    char buf[500];
    char *pos = buf;
    char *end = buf + sizeof(buf);
    int idx;
    struct rt_bandwidth *rt_b = sched_rt_bandwidth(rt_rq);

    pos += snprintf(pos, sizeof(buf),
        "sched: RT throttling activated for rt_rq %pK (cpu %d)\n",
        rt_rq, cpu_of(rq_of_rt_rq(rt_rq)));

    pos += snprintf(pos, end - pos,
        "rt_period_timer: expires=%lld now=%llu runtime=%llu period=%llu\n",
        hrtimer_get_expires_ns(&rt_b->rt_period_timer),
        ktime_get_ns(), sched_rt_runtime(rt_rq),
        sched_rt_period(rt_rq));

    if (bitmap_empty(array->bitmap, MAX_RT_PRIO))
        goto out;

// ......
out:
#ifdef CONFIG_PANIC_ON_RT_THROTTLING
/*
 * Use pr_err() in the BUG() case since printk_sched() will
 * not get flushed and deadlock is not a concern.
 */
    pr_err("%s\n", buf);
    BUG();
#else
    printk_deferred("%s\n", buf);
#endif

}
```

对应的汇编代码如下：

```Assembly
crash> dis -l dump_throttled_rt_tasks

// 截取展示重点信息
/home/dcbuild/anakin-workspace/anakin/anakin-la/lagvm/LINUX/android/kernel/msm-5.4/kernel/sched/rt.c: 967
0xffffffc01034a364 <dump_throttled_rt_tasks>:   str     x30, [x18], #8
0xffffffc01034a368 <dump_throttled_rt_tasks+4>: stp     x29, x30, [sp, #-96]!
0xffffffc01034a36c <dump_throttled_rt_tasks+8>: str     x28, [sp, #16]
0xffffffc01034a370 <dump_throttled_rt_tasks+12>:        stp     x26, x25, [sp, #32]
0xffffffc01034a374 <dump_throttled_rt_tasks+16>:        stp     x24, x23, [sp, #48]
0xffffffc01034a378 <dump_throttled_rt_tasks+20>:        stp     x22, x21, [sp, #64]
0xffffffc01034a37c <dump_throttled_rt_tasks+24>:        stp     x20, x19, [sp, #80]
0xffffffc01034a380 <dump_throttled_rt_tasks+28>:        mov     x29, sp
0xffffffc01034a384 <dump_throttled_rt_tasks+32>:        sub     sp, sp, #0x200
0xffffffc01034a388 <dump_throttled_rt_tasks+36>:        mov     x19, x0
/home/dcbuild/anakin-workspace/anakin/anakin-la/lagvm/LINUX/android/kernel/msm-5.4/kernel/sched/rt.c: 970
0xffffffc01034a38c <dump_throttled_rt_tasks+40>:        add     x0, sp, #0xc
0xffffffc01034a390 <dump_throttled_rt_tasks+44>:        mov     w2, #0x1f4                      // #500
0xffffffc01034a394 <dump_throttled_rt_tasks+48>:        mov     w1, wzr
0xffffffc01034a398 <dump_throttled_rt_tasks+52>:        mov     w22, #0x1f4                     // #500
0xffffffc01034a39c <dump_throttled_rt_tasks+56>:        add     x23, sp, #0xc
0xffffffc01034a3a0 <dump_throttled_rt_tasks+60>:        bl      0xffffffc0100b3340 <__efistub_memset>
```

**dump_throttled_rt_tasks** 函数具有一个参数，按 ARMv8 的 ABI 约定，通过 **x0** 寄存器传递相关参数。

上述汇编码显示，将x0的值传递给x19，我们将重点关注 **x19** 寄存器

从C源码可知，x19寄存器存放的值是一个地址，指向`struct rt_rq`类型的结构体

结合后续汇编码对 x19 的使用进行分析：

```C
/home/dcbuild/anakin-workspace/anakin/anakin-la/lagvm/LINUX/android/kernel/msm-5.4/kernel/sched/rt.c: 625
0xffffffc01034a3dc <dump_throttled_rt_tasks+120>:       ldr     x5, [x19, #1688]
```

此处将 **x19 + 1688** 的数值赋予 **x5**，对应源码：

```C
static inline u64 sched_rt_runtime(struct rt_rq *rt_rq)
{
    return rt_rq->rt_runtime;
}
```

即此处 **x19+1688** 的地址，存放 **rt_runtime** 数值。

我们可通过crash内置命令，输出结构体内部各个变量的偏移量

```C
crash> struct -o rt_rq
struct rt_rq {
     [0] struct rt_prio_array active;
  [1616] unsigned int rt_nr_running;
  [1620] unsigned int rr_nr_running;
         struct {
             int curr;
             int next;
  [1624] } highest_prio;
  [1632] unsigned long rt_nr_migratory;
  [1640] unsigned long rt_nr_total;
  [1648] int overloaded;
  [1656] struct plist_head pushable_tasks;
  [1672] int rt_queued;
  [1676] int rt_throttled;
  [1680] u64 rt_time;
  [1688] u64 rt_runtime;
  [1696] raw_spinlock_t rt_runtime_lock;
}
SIZE: 1768
```

显然，1688 偏移处即对应 **rt_runtime**，与之前的猜想一致。

尝试获取该结构体成员存储的数值：

```C
crash> rt_rq.rt_runtime ffffff8983caf480
  rt_runtime = 950000000,



// 或通过rd命令
crash> rd -o 1688 ffffff8983caf480 -64
ffffff8983cafb18:  00000000389fd980                    ...8....

crash> p 0x389fd980
$4 = 950000000
```

发现与崩溃log中runtime数值一致。至此，成功通过寄存器`x0`获取实际的函数参数数值。

## 解析位于函数栈的变量
一般crash显示的CPU寄存器数值，仅对最近一次的函数调用有效。若需要分析更为早期的函数参数，则需找到对应stack的地址，直接读取相应数值。

继续以此前的crash现场为例：
```C
PID: 15585    TASK: ffffff87fa61d980  CPU: 1    COMMAND: "droid.bluetooth"
 #0 [ffffffc01000b580] __delay at ffffffc0113da944
 #1 [ffffffc01000b5c0] __const_udelay at ffffffc0113daa18
 #2 [ffffffc01000b5e0] qcom_wdt_trigger_bite at ffffffc010940594
 #3 [ffffffc01000b600] do_vm_restart at ffffffc0101bba80
 #4 [ffffffc01000b620] atomic_notifier_call_chain at ffffffc01031f2c0
 #5 [ffffffc01000b660] machine_restart at ffffffc0102bac94
 #6 [ffffffc01000b780] panic at ffffffc0102e77ac
 #7 [ffffffc01000b830] die at ffffffc0102c4850
 #8 [ffffffc01000b860] bug_handler$387e226de7fcca202d1582dafa94315e at ffffffc0102c57c8
 #9 [ffffffc01000b890] brk_handler$04d00b2f5508655b50c86c3318e430f9 at ffffffc0102b7ec4
#10 [ffffffc01000b8b0] do_debug_exception at ffffffc0100a8974
#11 [ffffffc01000ba30] el1_dbg at ffffffc0100aa508
     PC: ffffffc01034a55c  [dump_throttled_rt_tasks+504]
     LR: ffffffc01034a55c  [dump_throttled_rt_tasks+504]
     SP: ffffffc01000ba40  PSTATE: 60400085
    X29: ffffffc01000bc40  X28: ffffff8983ca1bc0  X27: ffffffc012499240
    X26: ffffff8983caf240  X25: ffffff8983caf490  X24: ffffffc01000bc40
    X23: ffffff8983caf490  X22: ffffffc011d5485b  X21: ffffffc01000bb55
    X20: 0000000000000064  X19: ffffff8983caf480  X18: ffffffc01000d060
    X17: ffffff8227fe7e80  X16: 0000000000000280  X15: da2f3fd121cbcfbe
    X14: 0000000000000000  X13: 0000000000000000  X12: 0000000000000000
    X11: 00000000000000e0  X10: 0000000000000000   X9: e919127ca995d800
     X8: e919127ca995d800   X7: 0000000000000000   X6: ffffffc0103876dc
     X5: 0000000000000000   X4: 0000000000000080   X3: 0000000000000000
     X2: ffffff8983c9b318   X1: 0000000000000000   X0: 0000000000000109
#12 [ffffffc01000bc40] dump_throttled_rt_tasks at ffffffc01034a558
#13 [ffffffc01000bca0] update_curr_rt$c8e64db50a637c5c2b38997a086ae425 at ffffffc0103486b0
#14 [ffffffc01000bce0] task_tick_rt$c8e64db50a637c5c2b38997a086ae425 at ffffffc010347e98
#15 [ffffffc01000bd30] scheduler_tick at ffffffc01032ec98
#16 [ffffffc01000bd90] update_process_times at ffffffc0103b2fec
#17 [ffffffc01000bdc0] tick_sched_timer$2e93e54c57d54c141bd5e65a4951d56c at ffffffc0103cb148
```

为演示解析位于函数栈的变量方法，以#15栈帧为例。

由于此函数调用较早，当前的CPU寄存器值，与实际调用#15栈帧时已不一致，需在特定内存处寻找当时的值。

对应源码：
```C
void scheduler_tick(void)
{
    int cpu = smp_processor_id();
    struct rq *rq = cpu_rq(cpu);
    struct task_struct *curr = rq->curr;
    struct rq_flags rf;
    u64 wallclock;
    bool early_notif;
    u32 old_load;
    struct walt_related_thread_group *grp;
    unsigned int flag = 0;

    sched_clock_tick();

    rq_lock(rq, &rf);

    old_load = task_load(curr);
    set_window_start(rq);
    wallclock = sched_ktime_clock();
    walt_update_task_ravg(rq->curr, rq, TASK_UPDATE, wallclock, 0);
    update_rq_clock(rq);
    curr->sched_class->task_tick(rq, curr, 0);  // crash发生处
    calc_global_load_tick(rq);
    psi_task_tick(rq);
 // ...
 }
```

crash发生处的汇编代码如下：

```Assembly
/home/dcbuild/anakin-workspace/anakin/anakin-la/lagvm/LINUX/android/kernel/msm-5.4/kernel/sched/core.c: 3862
0xffffffc01032ec6c <scheduler_tick+172>:        ldr     x8, [x21, #144]
0xffffffc01032ec70 <scheduler_tick+176>:        adrp    x9, 0xffffffc01008f000 <__typeid__ZTSFiP11task_structPK11user_regsetjjPKvS5_E_global_addr>
0xffffffc01032ec74 <scheduler_tick+180>:        add     x9, x9, #0xba0
0xffffffc01032ec78 <scheduler_tick+184>:        ldr     x8, [x8, #136]
0xffffffc01032ec7c <scheduler_tick+188>:        sub     x9, x8, x9
0xffffffc01032ec80 <scheduler_tick+192>:        ror     x9, x9, #2
0xffffffc01032ec84 <scheduler_tick+196>:        cmp     x9, #0x18
0xffffffc01032ec88 <scheduler_tick+200>:        b.cs    0xffffffc01032ee84 <scheduler_tick+708>  // b.hs, b.nlast

// 传参
0xffffffc01032ec8c <scheduler_tick+204>:        mov     x0, x19
0xffffffc01032ec90 <scheduler_tick+208>:        mov     x1, x21
0xffffffc01032ec94 <scheduler_tick+212>:        mov     w2, wzr
0xffffffc01032ec98 <scheduler_tick+216>:        blr     x8
```

显然`x19->x0, x21->x1, wzr->w2`，正好对应下述函数的传入参数

```Assembly
 curr->sched_class->task_tick(rq, curr, 0);
```

以获取此处的rq实际参数值为例，需得到此刻的x19值。

为获取#15栈帧的**x19** 值，需分析下一层函数栈（`#14 [`**`ffffffc01000bce0`**`] task_tick_rt$c8e64db50a637c5c2b38997a086ae425`）：

```Assembly
crash> dis -l task_tick_rt$c8e64db50a637c5c2b38997a086ae425
/home/dcbuild/anakin-workspace/anakin/anakin-la/lagvm/LINUX/android/kernel/msm-5.4/kernel/sched/rt.c: 2733
0xffffffc010347e78 <task_tick_rt$c8e64db50a637c5c2b38997a086ae425>:     str     x30, [x18], #8
0xffffffc010347e7c <task_tick_rt$c8e64db50a637c5c2b38997a086ae425+4>:   stp     x29, x30, [sp, #-64]!
0xffffffc010347e80 <task_tick_rt$c8e64db50a637c5c2b38997a086ae425+8>:   stp     x24, x23, [sp, #16]
0xffffffc010347e84 <task_tick_rt$c8e64db50a637c5c2b38997a086ae425+12>:  stp     x22, x21, [sp, #32]
0xffffffc010347e88 <task_tick_rt$c8e64db50a637c5c2b38997a086ae425+16>:  stp     x20, x19, [sp, #48]
0xffffffc010347e8c <task_tick_rt$c8e64db50a637c5c2b38997a086ae425+20>:  mov     x29, sp
0xffffffc010347e90 <task_tick_rt$c8e64db50a637c5c2b38997a086ae425+24>:  mov     x20, x1
0xffffffc010347e94 <task_tick_rt$c8e64db50a637c5c2b38997a086ae425+28>:  mov     x19, x0
```

此处可见，**x19** 通过sp指针，保存在函数栈上。读取特定地址处的数值，可获得#15栈帧时的 **x19** 值

对应地址计算：
1. -64 + 48 + 8 = -8
2. 实际地址：**`ffffffc01000bce0-8=ffffffc01000bcd8`**

通过rd指令，读取该地址处存储的数值：

```Plain
crash> rd ffffffc01000bcd8
ffffffc01000bcd8:  ffffff8983caf240                    @.......
```

**ffffff8983caf240** 即为当时的 **x19** 寄存器存储的值，对应变量`struct rq *rq`

通过crash读取该结构体的值，发现数据显示正常，成功从函数栈上获取函数参数值：
```Shell
crash> rq ffffff8983caf240
struct rq {
  lock = {
    raw_lock = {
      {
        val = {
          counter = 257
        },
        {
          locked = 1 '\001',
          pending = 1 '\001'
        },
        {
          locked_pending = 257,
          tail = 0
        }
      }
    },
    magic = 3735899821,
    owner_cpu = 1,
    owner = 0xffffff87fa61d980,
    dep_map = {
      key = 0xffffffc012ad6b98 <sched_init..key>,
      class_cache = {0xffffffc012adb4d0 <lock_classes+8400>, 0xffffffc012ae3720 <lock_classes+41760>},
      name = 0xffffffc011c4c506 "&rq->lock",
      cpu = 1,
      ip = 18446743799103417368
    }
  },
  nr_running = 3,
  last_load_update_tick = 4294892296,
  last_blocked_load_update_tick = 4295081345,
  has_blocked_load = 1,
  nohz_tick_stopped = 0,
  nohz_flags = {
    counter = 0
  },
  nr_load_updates = 0,
  nr_switches = 2725652,
// ...
  hrtick_timer = {
    node = {
      node = {
        __rb_parent_color = 18446743564819562592,
        rb_right = 0x0,
        rb_left = 0x0
      },
      expires = 0
    },
    _softexpires = 0,
    function = 0xffffffc01009c494 <hrtick$9f3f973902601718a9a7a341611142c3.cfi_jt>,
    base = 0xffffff8983ac0bc0,
    state = 0 '\000',
    is_rel = 0 '\000',
    is_soft = 0 '\000',
    is_hard = 1 '\001',
    android_kabi_reserved1 = 0
  },
  rq_sched_info = {
    pcount = 1526865,
    run_delay = 47324308954,
    last_arrival = 0,
    last_queued = 0
  },
  rq_cpu_time = 205937214252,
  yld_count = 291674,
  sched_count = 3033800,
  sched_goidle = 1197984,
  ttwu_count = 1265985,
  ttwu_local = 204318,
  wake_list = {
    first = 0x0
  },
  idle_state = 0x0,
  idle_state_idx = 0
}
```