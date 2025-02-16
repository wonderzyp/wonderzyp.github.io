---
title: "案例: slab机制行为观测"
weight: 2
---


## 背景

系统层常出现内存问题，现观察内存子系统的slab机制行为，为后续阅读源码奠定基础。

本文将基于QEMU进行下述操作：
- 创建自定义的slab类型，提供分配、释放操作接口
- 操作上述接口，观察`/proc/slabinfo`的状态变化
- 开启`CONFIG_PAGE_OWNER`，观察slab与buddy system的关联

## 案例：创建自定义类型 kmem_cache
编译此内核模块：
```c
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/string.h>

#define CACHE_NAME "yep_slab_cache"
#define OBJECT_SIZE (4 * 1024)

static struct kmem_cache *my_cache;
static void **allocated_objects;
static int num_objects;
static struct kobject *my_kobj;
static int allocated_count;

static ssize_t alloc_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count)
{
    int i, ret;
    ret = kstrtoint(buf, 10, &num_objects);
    if (ret < 0)
        return ret;
    
    if (num_objects <= 0)
        return -EINVAL;
    
    allocated_objects = kmalloc_array(num_objects, sizeof(void *), GFP_KERNEL);
    if (!allocated_objects)
        return -ENOMEM;
    
    for (i = 0; i < num_objects; i++) {
        allocated_objects[i] = kmem_cache_alloc(my_cache, GFP_KERNEL);
        if (!allocated_objects[i]) {
            num_objects = i;
            break;
        }
    }
    allocated_count = num_objects;
    pr_info("Allocated %d objects from slab cache\n", num_objects);
    return count;
}

static ssize_t free_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count)
{
    int i;

    if (allocated_count <= 0) {
        pr_info("Nothing to free\n");
        return -1;
    }
    
    for (i = 0; i < allocated_count; i++) {
        if (allocated_objects[i]) {
            kmem_cache_free(my_cache, allocated_objects[i]);
            allocated_objects[i] = NULL;
        }
    }
    
    allocated_count = 0;

    pr_info("Freed all allocated objects from slab cache\n");
    return 0;
}


static struct kobj_attribute alloc_attribute = __ATTR_WO(alloc);
static struct kobj_attribute free_attribute = __ATTR_WO(free);

static int __init my_module_init(void)
{
    int retval;
    pr_info("Loading custom slab module...\n");
    
    my_cache = kmem_cache_create(CACHE_NAME, OBJECT_SIZE, 0, SLAB_HWCACHE_ALIGN, NULL);
    if (!my_cache) {
        pr_err("Failed to create slab cache\n");
        return -ENOMEM;
    }
    
    my_kobj = kobject_create_and_add("yep_slab", kernel_kobj);
    if (!my_kobj) {
        kmem_cache_destroy(my_cache);
        return -ENOMEM;
    }
    
    retval = sysfs_create_file(my_kobj, &alloc_attribute.attr);
    if (retval) goto error;
    retval = sysfs_create_file(my_kobj, &free_attribute.attr);
    if (retval) goto error;
    
    pr_info("Slab cache created and sysfs interface added\n");
    return 0;

error:
    kobject_put(my_kobj);
    kmem_cache_destroy(my_cache);
    return retval;
}

static void __exit my_module_exit(void)
{
    int i;
    if (allocated_objects) {
        for (i = 0; i < allocated_count; i++) {
            if (allocated_objects[i])
                kmem_cache_free(my_cache, allocated_objects[i]);
        }
        kfree(allocated_objects);
    }
    
    if (my_cache) {
        kmem_cache_destroy(my_cache);
        pr_info("Slab cache destroyed\n");
    }
    
    kobject_put(my_kobj);
    pr_info("Custom slab module unloaded\n");
}

module_init(my_module_init);
module_exit(my_module_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("yep");
MODULE_DESCRIPTION("A kernel module that creates a custom slab cache");

```
上述demo将创建自定义的slab(yep_slab_cache)，大小为4*1024 bytes，并基于sysfs提供申请与释放的接口。

### 申请自定义slab并观察slabinfo数据

```Bash
# insmod yep_slab.ko

# cat /proc/slabinfo
slabinfo - version: 2.1
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
yep_slab_cache         0      0   4544    7    8 : tunables    0    0    0 : slabdata      0      0      0
```

上述输出描述yep_slab_cache状态：
- 目前共0个obj，已使用0个obj
- 每个obj大小为4544bytes：声明为4096bytes，但包含一些额外的元信息数据
- 每个slab由8个page组成
- 每个slab最多可容纳7个objs：`[8*4096/4544] = 7`
- 目前有0个slab处于活跃态，共0个slab

申请此slab：
```Bash
# echo 1 > /sys/kernel/yep_slab/alloc
[  518.353944] Allocated 1 objects from slab cache
#
# cat /proc/slabinfo |grep yep
yep_slab_cache         1      7   4544    7    8 : tunables    0    0    0 : slabdata      1      1      0
```
结果展示：
- active_objs数量增加至1, num_objs增加至7
- active_slabs与num_slabs均增加1

申请时以slab为基本单位增减。由于每个自定义slab可容纳7个objs，即使申请1个obj, 还会有6个冗余的objs待使用。只有当7个objs均被使用后，才会与buddy system互动，申请新的物理页。  

例如，再申请6个objs，依旧在当前slab内申请，不涉及物理页操作。

```Bash
# echo 6 > /sys/kernel/yep_slab/alloc
[  305.574849] Allocated 6 objects from slab cache
# cat /proc/slabinfo | grep yep
yep_slab_cache         7      7   4544    7    8 : tunables    0    0    0 : slabdata      1      1      0
```

此时yep_slab_cache无剩余可用objs，再次申请时将创建新的slab，该操作伴随物理页的申请。

```Bash
# echo 1 > /sys/kernel/yep_slab/alloc
[  310.657329] Allocated 1 objects from slab cache
# cat /proc/slabinfo | grep yep
yep_slab_cache         8     14   4544    7    8 : tunables    0    0    0 : slabdata      2      2      0
```

注意此处的num_slabs已增长至2.

显然，通过上述方法，可避免每次申请obj时频繁地直接操作内存，降低性能损耗。

### 追踪buddy system使用情况
上述流程通过`/proc/slabinfo`的active_slabs与num_slabs列信息判断与buddy system的互动。

为了更加明确的展示信息，可开启`CONFIG_PAGE_OWNER`配置项，并在QEMU传入`page_owner=on`启动参数

挂载debugfs

```Bash
# mkdir /d
# mount -t debugfs none /d
```

刚加载此内核模块时，由于未申请objs，无实际对应的page分配信息。

```Bash
# cat /proc/slabinfo  | grep yep
yep_slab_cache         0      0   4544    7    8 : tunables    0    0    0 : slabdata      0      0      0
#
# cat /d/page_owner | grep yep
#
```
申请一个obj后，会分配实际的物理page，且page_owner下有相应信息：
```Bash
# echo 1 > /sys/kernel/yep_slab/alloc
[  546.264589] Allocated 1 objects from slab cache
yep_slab_cache         1      7   4544    7    8 : tunables    0    0    0 : slabdata      1      1      0

# cat /d/page_owner | grep yep -A 10 -B 10
Page allocated via order 3, mask 0xd20c0(__GFP_IO|__GFP_FS|__GFP_NOWARN|__GFP_NORETRY|__GFP_COMP|__GFP_NOMEMALLOC), pid 158, ts 546264411584 ns, free_ts 1667743600 ns
PFN 284392 type Unmovable Block 555 type Unmovable Flags 0x3fffc0000010200(slab|head|node=0|zone=0|lastcpupid=0xffff)
 post_alloc_hook+0x124/0x128
 prep_new_page+0x34/0xf8
 get_page_from_freelist+0x1308/0x134c
 __alloc_pages+0x14c/0x2e8
 alloc_pages+0x1d8/0x2cc
 new_slab+0xd0/0x5ac
 ___slab_alloc+0x2a8/0x66c
 kmem_cache_alloc+0x2b0/0x3a0
 alloc_store+0xb8/0xfc [yep_slab]
 kobj_attr_store+0x1c/0x30
 sysfs_kf_write+0x48/0x5c
 kernfs_fop_write_iter+0xe4/0x184
 vfs_write+0x2cc/0x400
 ksys_write+0x84/0xf0
 __arm64_sys_write+0x28/0x34
 invoke_syscall+0x4c/0x10c
```

同样的，后续若继续申请objs，num_slabs数值与page_owner记录的yep_slab数量一致：

```Bash
# cat /proc/slabinfo | grep yep_
yep_slab_cache        60     63   4544    7    8 : tunables    0    0    0 : slabdata      9      9      0

对应page_owner记录9次
# cat /d/page_owner | grep yep
 alloc_store+0xb8/0xfc [yep_slab]
 alloc_store+0xb8/0xfc [yep_slab]
 alloc_store+0xb8/0xfc [yep_slab]
 alloc_store+0xb8/0xfc [yep_slab]
 alloc_store+0xb8/0xfc [yep_slab]
 alloc_store+0xb8/0xfc [yep_slab]
 alloc_store+0xb8/0xfc [yep_slab]
 alloc_store+0xb8/0xfc [yep_slab]
 alloc_store+0xb8/0xfc [yep_slab]
#
```