---
title: "调度 schedule"
# weight: 2
---

## 概述

本章节分析调度的核心函数`schedule()`, 简化后的关键逻辑如下:

![main_of_schedule.png](/docs/kernel/subsystem/schedule/assert/schedule.png)

```c
asmlinkage __visible void __sched schedule(void)
{
	struct task_struct *tsk = current;

	sched_submit_work(tsk);
	do {
		preempt_disable();
		__schedule(SM_NONE); // the main scheduler function
		sched_preempt_enable_no_resched();
	} while (need_resched()); // test the TIF_NEED_RESCHED flag
	sched_update_worker(tsk);
}
EXPORT_SYMBOL(schedule);
```

## need_resched: 检测TIF_NEED_RESCHED标志位

`need_resched()`检测`struct thread_info`的flag成员, 判断`TIF_NEED_RESCHED`是否置位
```c
// 最终执行下述代码
#define tif_need_resched() test_thread_flag(TIF_NEED_RESCHED)

static inline int test_ti_thread_flag(struct thread_info *ti, int flag)
{
	return test_bit(flag, (unsigned long *)&ti->flags);
}

#define TIF_NEED_RESCHED	1	/* rescheduling necessary */
```

## __schedule

`__schedule`为调度的核心部分, 主要流程如下:

![__schedule.png](/docs/kernel/subsystem/schedule/assert/__schedule.png)

从kernel的源码注释可知, 下述场景将调用此函数:

1. 显式的阻塞：mutex, semaphore, waitqueue
2. `TIF_NEED_RESCHED`，表明此任务需要被调度，内核会定期检测此标志位：
    - interrupt、userspace return paths
    - 该标志位会被`scheduler_tick()`函数周期性**设置**
注: Wakeups 不会实际调用`schedule()`，仅将task加入runqueue


该函数关键在于:
- 选取待被调度任务next: `pick_next_task(rq, prev, &rf)`
- 清空调度位: `clear_tsk_need_resched(prev)`
- 使用next抢占curr: `context_switch(rq, prev, next, &rf)`


### pick_next_task
`pick_next_task`选取最适合的task, 不同调度类拥有各自的实现方式.

以CFS为例, 最终通过`pick_next_entity`获取最优待调度的se.
选取逻辑包含:
- 最小vruntime: 借助rbtree
- 特殊处理`cfs_rq->skip`
- 若`cfs_rq->next`存在, 优先调度
- 否则, 优先`cfs_rq->last`, 充分利用局部缓存?

获取se后,调用`set_next_entity(cfs_rq, se)`,设置`cfs_rq->curr = se`

### context_switch

任务上下文切换包含两核心步骤：
- 保存当前进程状态：
    - CPU state and stack
    - task的`struct mm_struct *mm`或`struct mm_struct *active_mm`
- 恢复下一个进程状态


### 切换MM成员

内核态下，没有`mm`成员，仅有`active_mm`
用户态下, `mm`与`active_mm`一致
内核态下, `mm`为空, `active_mm`为地址空间

按切换前后的状态, 包含以下四类:
- from kernel to kernel: 切换active_mm, 将prev->active_mm置空
- from user to kernel: 切换active_mm, 增加`prev->active_mm`计数

- from kernel to user: `rq->prev_mm = prev->active_mm`, 并将`prev->active_mm`置为空
- from user to user: 


### 切换寄存器状态与栈
调用`cpu_switch_to`汇编码，切换寄存器与堆栈信息
```asm
/*
 * Register switch for AArch64. The callee-saved registers need to be saved
 * and restored. On entry:
 *   x0 = previous task_struct (must be preserved across the switch)
 *   x1 = next task_struct
 * Previous and next are guaranteed not to be the same.
 *
 */
SYM_FUNC_START(cpu_switch_to)
	// x0 当前任务的task_struct指针
	// x10 表明当前任务保存寄存器的上下文偏移量
	mov	x10, #THREAD_CPU_CONTEXT
	add	x8, x0, x10
	mov	x9, sp

	// x19-x28为被调用者保存的寄存器
	// 任务切换时，需保存到当前任务的上下文区域
	stp	x19, x20, [x8], #16		// store callee-saved registers
	stp	x21, x22, [x8], #16
	stp	x23, x24, [x8], #16
	stp	x25, x26, [x8], #16
	stp	x27, x28, [x8], #16
	// x29: fp指针，指向帧栈的顶部，记录当时栈的起始位置
	// x9: sp，指向栈当前位置
	// lr: 链接寄存器，保存函数返回位置
	stp	x29, x9, [x8], #16
	str	lr, [x8]

	// x1为next任务的task_struct指针
	add	x8, x1, x10
	// 恢复next任务保存的上下文
	ldp	x19, x20, [x8], #16		// restore callee-saved registers
	ldp	x21, x22, [x8], #16
	ldp	x23, x24, [x8], #16
	ldp	x25, x26, [x8], #16
	ldp	x27, x28, [x8], #16
	ldp	x29, x9, [x8], #16
	ldr	lr, [x8]

	// 切换栈指针，将sp置为下一任务的栈顶
	mov	sp, x9
	// 此时，CPU将在新任务的堆栈运行
	
	msr	sp_el0, x1 // 设置用户态的栈指针，用于支持从内核态返回至用户态

	// 支持ARM的特性
	ptrauth_keys_install_kernel x1, x8, x9, x10
	scs_save x0
	scs_load_current
	ret  // 返回新的任务上下文
SYM_FUNC_END(cpu_switch_to)
NOKPROBE(cpu_switch_to)
```

上述代码实现CPU寄存器的保存与切换:
- 将prev的x19-x28, fp, sp, lr存储于prev的`task_struct.thread.cpu_context`内
- 读取next的`task_struct.thread.cpu_context`，并将数值设置到寄存器上

最终，通过`mov sp, x9`，将sp指针设置为next中存储的值，完成栈指针sp的切换.
