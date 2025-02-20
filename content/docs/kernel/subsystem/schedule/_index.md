---
weight: 2
bookCollapseSection: true
bookToc: true
title: "Schedule"
---

## 调度子系统总述

调度子系统负责分配CPU运行资源（即每个task可占用CPU的时间），需确定以下信息：
- 调度执行哪一个task
- 何时开始调度
- task的执行时长

### 如何选择待调度的task

不同种类task需不同的调度策略，考虑两种不同的使用场景：
- I/O消耗型：输入字符时，用户希望按下按键能被立刻响应。
- CPU消耗型：解算CFD状态方程等，及时响应性不重要（但尽快执行完成依旧很重要）

为适应不同种类task的调度需求，kernel以模块化的形式，提供CFS, RT, DL等**调度类(sched_class)**。
各个调度类实现自身独特的调度逻辑，其中 **CFS(Complete Fair Scheduler)** 最为普遍。

CFS调度策略侧重于公平性，每个task维护自身的已运行时长runtime。调度时将基于内核的红黑树数据结构，选择runtime最小的task，作为下一个待执行的task。

考虑在实际系统中各个task优先级不同，CFS进一步引入**虚拟时间vruntime**的概念。

CFS按task的优先级权重，对runtime作伸缩变换。设可分配的实际CPU墙上时间(wall time)为100ms。若两个任务的权重为7:3，则100ms将均分为70ms与30ms。
若将实际分得的wall time处以各自的比例，则均能归一化为100ms:
- 70/(7/10)=100
- 30/(3/10)=100

这种归一化后的时间，即为vruntime。

如此，对于不同优先级的task，可直接比较并选择最小的vruntime，以确定下一个待执行的task。

> 可以理解成**不同优先级task的时钟流速不同**，优先级越高，时钟走得越慢。


### 何时开始调度
不同调度类拥有独特的调度策略，此处依旧以CFS为例。

CFS的基本原则是尽量保证各个task的vruntime相等。为实现此目的，当正在运行task的vruntime不再是runqueue内最小时，调度器将其放回runqueue内，并选择拥有最小vruntime的task开始运行。

此外，并非所有种类的task都希望得到调度。例如某个task正在等待I/O操作完成，提前调度毫无作用且浪费CPU资源。

对此，内核为每个task维护当前的状态信息，CFS仅会调度处于`TASK_RUNNING`态的task。
> 上述等待I/O操作的task，一般处于休眠状态，对应`TASK_INTERRUPTIBLE`或`TASK_UNINTERRUPTIBLE`。
> 当等待的事件完成时，会转化为`TASK_RUNNING`

关于CFS的更多设计细节：
- 需保证每个task在间隔一段时间后，均能执行一次。
- 需避免task不会太快地被调度，避免频繁上下文切换带来的性能损耗。
- 引入**组调度**，避免多线程的进程因较多的线程数量，得到较多的执行时间。


### 持续执行的时间长度

从前文的描述来看，CFS的调度逻辑不包含传统的时间片概念。其通过维护各个task的vruntime尽可能一致，使得各个task齐头并进，体现其公平性。

> 时间片描述当前task被抢占前，可持续执行的时间。
> 时间片的长度直接影响系统性能，过短时间片将导致频繁调度，增加性能损耗；过长的时间片将导致系统响应不及时。