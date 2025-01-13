---
title: "运行队列 runqueue"
# weight: 2
---

调度器的最终目的，是将所有可执行task在CPU上运行
kernel内使用runqueue存储就绪态任务，`struct rq`结构体内的关键成员如下：
```c
struct rq {
  unsigned int	nr_running;
  
  struct cfs_rq	cfs;	// CFS runqueue 实现
  struct rt_rq	rt;		// RT runqueue 实现
  struct dl_rq	dl;		// Deadline runqueue 实现

  struct task_struct __rcu	*curr;	// 当前runqueue中正在运行的task
  struct task_struct				*idle;	// Idle 调度类任务
  struct task_struct 				*stop;	// Stop 调度类任务 
}
```

runqueue存储可执行的task，调度器的大多行为围绕rq进行

此外，每个CPU拥有各自的rq，可避免多CPU同时访问相同rq


## struct cfs_rq
以`struct cfs_rq`为例, 进一步观察内部成员定义:
```c
struct cfs_rq {
    struct sched_entity	*curr;
    struct sched_entity	*next;
    struct sched_entity	*last;
    struct sched_entity	*skip;
}
```

## 实例
crash工具内置`runq`命令，可获取所有CPU核心上的runqueue状态：
```bash
crash> runq
CPU 0 RUNQUEUE: ffffff8998964500
  CURRENT: PID: 0      TASK: ffffffc00adee100  COMMAND: "swapper/0"
  RT PRIO_ARRAY: ffffff8998964880
     [no tasks queued]
  CFS RB_ROOT: ffffff8998964730
     [no tasks queued]

CPU 1 RUNQUEUE: ffffff8998981500
  CURRENT: PID: 0      TASK: ffffff81b5f92400  COMMAND: "swapper/1"
  RT PRIO_ARRAY: ffffff8998981880
     [ 98] PID: 14     TASK: ffffff81b5e7b600  COMMAND: "rcu_preempt"
  CFS RB_ROOT: ffffff8998981730
     [120] PID: 4603   TASK: ffffff8223072400  COMMAND: "ToxCoreThread"

CPU 2 RUNQUEUE: ffffff899899e500
  CURRENT: PID: 17340  TASK: ffffff89559eda00  COMMAND: "Profile Saver"
  RT PRIO_ARRAY: ffffff899899e880
     [no tasks queued]
  CFS RB_ROOT: ffffff899899e730
     [120] PID: 3110   TASK: ffffff8193f5da00  COMMAND: "ToxCoreThread"
     [120] PID: 2958   TASK: ffffff8179652400  COMMAND: "ToxCoreThread"
     [120] PID: 4854   TASK: ffffff81e8225a00  COMMAND: "ToxCoreThread"
     [120] PID: 1466   TASK: ffffff80e4f5b600  COMMAND: "ToxCoreThread"

CPU 3 RUNQUEUE: ffffff89989bb500
  CURRENT: PID: 0      TASK: ffffff81b5f90000  COMMAND: "swapper/3"
  RT PRIO_ARRAY: ffffff89989bb880
     [no tasks queued]
  CFS RB_ROOT: ffffff89989bb730
     [no tasks queued]
```
上述结果揭示，每个CPU会单独维护各自rq，且不同调度类存在不同rq


