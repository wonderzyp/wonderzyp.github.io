---
title: "调度类 sched_class"
# weight: 2
---

为提升调度器的拓展性，内核使用`struct sched_class`模块化调度器
结构体内定义大量通用接口，各调度器按需自行实现

## sched_class定义与初始化

`struct sched_class`关键成员定义如下：
```c
struct sched_class {
  // 向自定义调度器的runqueue内添加、删除进程
  void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
  void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);

  // 新建或唤醒进程时，检查该进程能否抢占CPU
  void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags);

  // 从rq内选择最合适的task
  struct task_struct *(*pick_next_task)(struct rq *rq);
}
```

### 各种sched_class的初始化
以Kernel 5.15.163为例，支持的调度类定义如下：
```c
#define SCHED_DATA				\
	STRUCT_ALIGN();				\
	__begin_sched_classes = .;		\
	*(__idle_sched_class)			\
	*(__fair_sched_class)			\
	*(__rt_sched_class)			\
	*(__dl_sched_class)			\
	*(__stop_sched_class)			\
	__end_sched_classes = .;

// 对应场景如下
__fair_sched_class  // CFS  
__rt_sched_class    // 实时调度类，确保在特定时间窗口内获得CPU
__dl_sched_class    // 基于截至时间的调度
__stop_sched_class  // 停机任务的调度，如CPU热插拔
```

上述`sched_class`名称由下述宏拼接:
```c
// sched.h 内定义相关宏
#define DEFINE_SCHED_CLASS(name) \
const struct sched_class name##_sched_class \
	__aligned(__alignof__(struct sched_class)) \
	__section("__" #name "_sched_class")

// 不同sched_class使用上述宏，定义不同名称
// 且定义时绑定不同的实现方法
// 例如：
DEFINE_SCHED_CLASS(fair) = {
  .enqueue_task		= enqueue_task_fair,
  .dequeue_task		= dequeue_task_fair,
  .yield_task		= yield_task_fair,
  .yield_to_task		= yield_to_task_fair,
  .check_preempt_curr = check_preempt_wakeup,
  // ...
}
```

不同的`sched_class`拥有各自的调度策略
- DL: Earliest Deadline First，对多个DL进程调度，靠前的优先执行
- RT: 选最高优先级的执行，范围`[1, 99]`，其中99最高
- CFS: 进程的nice值

## sched_class 函数接口作用（以CFS为例）

主要分析CFS关键接口：
```c
DEFINE_SCHED_CLASS(fair) = {
  .enqueue_task = enqueue_task_fair,
  .dequeue_task = dequeue_task_fair,
  .check_preempt_curr = check_preempt_wakeup,
  .pick_next_task = __pick_next_task_fair,
}
```

### check_preempt_wakeup
该函数检测当前进程能否抢占

唤醒新进程时，需检测抢占。新进程可能具有更高优先级或更小vruntime
```c
check_preempt_curr(rq, p, WF_FORK)
```

1. 若唤醒进程与当前进程属于相同调度类，直接调用该调度类的`check_preempt_curr`函数
2. 若不属于相同调度类，需比较调度类优先级


能否抢占以curr与se的`vruntime`相对大小关系，作为判断依据：
- curr < se：当前运行se的vruntime 小于 新建se的vruntime，无法抢占，返回-1
- curr > se 但差值小于gran：不抢占，返回0
- curr > se 且差值大于gram：抢占

总结: 新建进程vruntime需小于curr的vruntime, 且需大于gran

> gran为唤醒抢占粒度，避免抢占过于频繁。
> gran对应变量sysctl_sched_wakeup_granularity，默认为1ms

以下为对应源码：
```c
/*
 * Should 'se' preempt 'curr'.
 *
 *             |s1
 *        |s2
 *   |s3
 *         g
 *      |<--->|c
 *
 *  w(c, s1) = -1
 *  w(c, s2) =  0
 *  w(c, s3) =  1
 *
 */
static int
wakeup_preempt_entity(struct sched_entity *curr, struct sched_entity *se)
{
	s64 gran, vdiff = curr->vruntime - se->vruntime;

	if (vdiff <= 0)
		return -1;

	gran = wakeup_gran(se);  // unsigned int sysctl_sched_wakeup_granularity = 1000000UL; 1ms
	if (vdiff > gran)
		return 1;

	return 0;
}
```

满足抢占条件时，设置`TIF_NEED_RESCHED`的标志位：
```c
static inline void set_tsk_need_resched(struct task_struct *tsk)
{
	set_tsk_thread_flag(tsk,TIF_NEED_RESCHED);
}
```


