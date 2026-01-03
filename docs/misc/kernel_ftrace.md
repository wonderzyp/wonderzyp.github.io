---
title: "Kernel ftrace 机制的使能与使用"
weight: 2
---

## 背景
调试内核相关功能时，多涉及追踪函数调用、记录函数耗时等需求。传统方式可在内核函数处手动添加计时或`printk`等手段追踪，但较为繁琐。

鉴于内核内置的ftrace机制可支持此类需求，本文以追踪`__arm64_sys_umount`函数为例， 介绍function与function_graph机制，实现函数调用与函数耗时的追踪。

## 编译使能
需开启以下Kconfig
```text
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y
CONFIG_DYNAMIC_FTRACE=y
CONFIG_DYNAMIC_FTRACE_WITH_REGS=y
```

## 调试步骤

### 挂载tracefs
```text
// 挂载tracefs
mkdir t
mount -t tracefs none ./t
cd ./t

# cat available_tracers
function_graph function nop
// 确保有function与function_graph俩功能
```

### 追踪函数调用: function
```text
# echo function > current_tracer
# echo __arm64_sys_umount > set_ftrace_filter
# echo > trace

// 手动制造一个umount事件
# mkdir ~/tmp_dir
# mount -t tracefs none ~/tmp_fs
# umount ~/tmp_fs

// 此时可见函数调用情况
# cat trace  
# tracer: function  
#  
# entries-in-buffer/entries-written: 1/1   #P:4  
#  
#                                _-----=> irqs-off  
#                               / _----=> need-resched  
#                              | / _---=> hardirq/softirq  
#                              || / _--=> preempt-depth  
#                              ||| / _-=> migrate-disable  
#                              |||| /     delay  
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION  
#              | |         |   |||||     |         |  
         umount-219     [003] .....   777.540600: __arm64_sys_umount <-invoke_syscall
```

### 记录函数耗时: function_graph
操作与`function`基本类似，但此tracer侧重追踪函数耗时

```text
# echo function_graph > current_tracer
# echo __arm64_sys_umount > set_ftrace_filter
# echo > trace

// trigger umount
# mount -t tracefs none ~/tmp_fs  
# umount ~/tmp_fs

// 查看耗时记录
# cat trace  
# tracer: function_graph  
#  
# CPU  DURATION                  FUNCTION CALLS  
# |     |   |                     |   |   |   |  
1) ! 521.776 us  |  __arm64_sys_umount();
```
若需更详细的调用栈上耗时，可设置`set_graph_function`
```text
// trigger umount
# mount -t tracefs none ~/tmp_fs  
# umount ~/tmp_fs

# cat trace  
# tracer: function_graph  
#  
# CPU  DURATION                  FUNCTION CALLS  
# |     |   |                     |   |   |   |  
3)               |  __arm64_sys_umount() {  
3)               |    user_path_at_empty() {  
3)               |      getname_flags() {  
3)               |        kmem_cache_alloc() {  
3)   2.400 us    |          should_failslab();  
3) + 16.928 us   |        }  
3) + 20.864 us   |      }  
3)               |      filename_lookup() {  
3)               |        path_lookupat() {  
3)               |          path_init() {  
3)   1.600 us    |            __rcu_read_lock();  
3)               |            nd_jump_root() {  
3)   1.632 us    |              set_root();  
3)   4.880 us    |            }  
3) + 11.184 us   |          }  
3)               |          link_path_walk() {  
3)               |            inode_permission() {  
3)               |              generic_permission() {  
3)               |                check_acl() {  
3)   1.584 us    |                  get_cached_acl_rcu();  
3)   4.656 us    |                }  
3)   7.728 us    |              }  
3)   1.616 us    |              security_inode_permission();  
3) + 13.728 us   |            }  
3)               |            walk_component() {  
3)               |              lookup_fast() {  
3)   1.568 us    |                __d_lookup_rcu();  
3)   4.624 us    |              }  
3)   1.760 us    |              step_into();  
3) + 10.944 us   |            }  
3)               |            inode_permission() {  
3)               |              generic_permission() {  
3)               |                check_acl() {  
3)   1.456 us    |                  get_cached_acl_rcu();  
3)   4.352 us    |                }  
3)   7.280 us    |              }  
3)   1.568 us    |              security_inode_permission();  
3) + 13.248 us   |            }  
3) + 43.824 us   |          }  
3)               |          walk_component() {  
3)               |            lookup_fast() {  
3)   1.552 us    |              __d_lookup_rcu();  
3)   4.608 us    |            }  
3)               |            step_into() {  
3)   1.600 us    |              __lookup_mnt();  
3)   5.024 us    |            }  
3) + 14.064 us   |          }  
3)               |          handle_lookup_down() {  
3)   1.824 us    |            step_into();  
3) + 13.424 us   |          }  
3)               |          complete_walk() {  
3)               |            try_to_unlazy() {  
3)   1.568 us    |              legitimize_links();  
3)   1.856 us    |              __legitimize_mnt();  
3)   1.648 us    |              __rcu_read_unlock();  
3) + 11.200 us   |            }  
3) + 14.272 us   |          }  
3)               |          terminate_walk() {  
3)   1.568 us    |            dput();  
3)   1.568 us    |            mntput();  
3)   7.808 us    |          }  
3) ! 118.432 us  |        }  
3) ! 121.856 us  |      }  
3)   1.840 us    |      kmem_cache_free();  
3) ! 151.984 us  |    }  
3)               |    path_umount() {  
3)               |      ns_capable() {  
3)               |        security_capable() {  
3)   1.744 us    |          cap_capable();  
3)   5.680 us    |        }  
3)   9.744 us    |      }  
3) + 12.448 us   |      security_sb_umount();  
3)   1.728 us    |      down_write();  
3)   1.744 us    |      _raw_spin_lock();  
3)               |      propagate_mount_busy() {  
3) + 14.112 us   |        mnt_get_count();  
3) + 30.144 us   |      }  
3)               |      umount_tree() {  
3) + 10.912 us   |        propagate_mount_unlock();  
3) + 26.160 us   |        propagate_umount();  
3)               |        __wake_up() {  
3)   1.920 us    |          _raw_spin_lock_irqsave();  
3)   1.600 us    |          __wake_up_common();  
3)   1.680 us    |          _raw_spin_unlock_irqrestore();  
3) + 12.352 us   |        }  
3)               |        umount_mnt() {  
3)   1.728 us    |          _raw_spin_lock();  
3)   1.568 us    |          _raw_spin_unlock();  
3)               |          dput_to_list() {  
3)   1.456 us    |            __rcu_read_lock();  
3)   9.040 us    |            __rcu_read_unlock();  
3) + 16.176 us   |          }  
3)   2.448 us    |          kfree();  
3) + 39.344 us   |        }  
3) + 14.064 us   |        change_mnt_propagation();  
3) ! 142.704 us  |      }  
3)   1.808 us    |      _raw_spin_unlock();  
3)               |      namespace_unlock() {  
3)   1.808 us    |        up_write();  
3)   2.592 us    |        shrink_dentry_list();  
3)               |        synchronize_rcu_expedited() {  
3) + 13.264 us   |          rcu_gp_is_normal();  
3)   2.128 us    |          mutex_trylock();  
3)   9.440 us    |          sync_exp_work_done();  
3)               |          __init_work() {  
3)   2.720 us    |            _raw_spin_lock_irqsave();  
3)   1.680 us    |            _raw_spin_unlock_irqrestore();  
3) + 19.024 us   |          }  
3)               |          queue_work_on() {  
3)               |            __queue_work() {  
3)   1.552 us    |              __rcu_read_lock();  
3)   1.952 us    |              _raw_spin_lock();  
3)   1.632 us    |              _raw_spin_lock_irqsave();  
3)   1.520 us    |              _raw_spin_unlock_irqrestore();  
3)               |              wake_up_process() {  
3)               |                try_to_wake_up() {  
3)   2.016 us    |                  _raw_spin_lock_irqsave();  
3)   2.256 us    |                  kthread_is_per_cpu();  
3)   1.712 us    |                  ttwu_queue_wakelist();  
3)   1.520 us    |                  _raw_spin_lock();  
3)               |                  update_rq_clock() {  
3)   1.728 us    |                    update_irq_load_avg();  
3)   5.248 us    |                  }  
3)               |                  ttwu_do_activate() {  
3)               |                    enqueue_task_fair() {  
3)               |                      update_curr() {  
3)   1.600 us    |                        __calc_delta();  
3)   5.392 us    |                      }  
3)   1.824 us    |                      __update_load_avg_se();  
3)   1.680 us    |                      __update_load_avg_cfs_rq();  
3) + 16.928 us   |                    }  
3)               |                    ttwu_do_wakeup() {  
3)               |                      check_preempt_curr() {  
3)               |                        check_preempt_wakeup() {  
3)   1.472 us    |                          update_curr();  
3)   2.160 us    |                          set_next_buddy();  
3)   1.920 us    |                          resched_curr();  
3) + 13.216 us   |                        }  
3) + 16.560 us   |                      }  
3) + 24.432 us   |                    }  
3) + 47.408 us   |                  }  
3)   1.520 us    |                  _raw_spin_unlock();  
3)   1.520 us    |                  _raw_spin_unlock_irqrestore();  
3) + 78.752 us   |                }  
3) + 82.032 us   |              }  
3)   2.464 us    |              _raw_spin_unlock();  
3) + 16.288 us   |              __rcu_read_unlock();  
3) ! 129.856 us  |            }  
3)               |          rcu_note_context_switch() {  
3)   1.616 us    |            rcu_qs();  
3)   5.536 us    |          }  
3)   1.552 us    |          _raw_spin_lock();  
3)   1.536 us    |          update_rq_clock();  
3)               |          pick_next_task_fair() {  
3)   1.424 us    |            update_curr();  
3)   1.744 us    |            pick_next_entity();  
3)               |            put_prev_entity() {  
3)               |              update_curr() {  
3)   1.504 us    |                cpuacct_charge();  
3)   4.560 us    |              }  
3)   1.744 us    |              __update_load_avg_se();  
3)   1.568 us    |              __update_load_avg_cfs_rq();  
3) + 15.360 us   |            }  
3)               |            put_prev_entity() {  
3)   1.440 us    |              update_curr();  
3)   1.568 us    |              __update_load_avg_se();  
3)   1.488 us    |              __update_load_avg_cfs_rq();  
3) + 10.688 us   |            }  
3)               |            set_next_entity() {  
3)   3.392 us    |              __update_load_avg_se();  
3)   1.504 us    |              __update_load_avg_cfs_rq();  
3) + 10.256 us   |            }  
3) + 50.192 us   |          }  
3)               |          fpsimd_thread_switch() {  
3)   2.768 us    |            fpsimd_save();  
3)   6.672 us    |          }  
3)   1.680 us    |          hw_breakpoint_thread_switch();  
3)               |          this_cpu_has_cap() {  
3)   2.176 us    |            is_affected_midr_range_list();  
3)   5.632 us    |          }  
3)   1.552 us    |          mte_thread_switch();  
3)               |          finish_task_switch() {  
3)   1.616 us    |            _raw_spin_unlock();  
3)   5.984 us    |          }  
3) ! 133.520 us  |          } /* queue_work_on */  
3)   2.320 us    |          sync_exp_work_done();  
3)   1.648 us    |          mutex_unlock();  
3)               |          destroy_work_on_stack() {  
3)   1.840 us    |            _raw_spin_lock_irqsave();  
3)   1.600 us    |            _raw_spin_unlock_irqrestore();  
3) + 15.520 us   |          }  
3) ! 457.904 us  |        }  
3)               |        mntput_no_expire() {  
3)   1.520 us    |          __rcu_read_lock();  
3)   1.744 us    |          _raw_spin_lock();  
3)   2.928 us    |          mnt_get_count();  
3)   1.568 us    |          __rcu_read_unlock();  
3)   1.632 us    |          _raw_spin_unlock();  
3) + 19.920 us   |        }  
3) ! 494.896 us  |      }  
3)               |      dput() {  
3)   1.472 us    |        __rcu_read_lock();  
3)   1.552 us    |        __rcu_read_unlock();  
3)   8.304 us    |      }  
3)               |      mntput_no_expire() {  
3)   1.456 us    |        __rcu_read_lock();  
3)   1.712 us    |        _raw_spin_lock();
3)   1.952 us    |        mnt_get_count();
3)   1.520 us    |        __rcu_read_unlock();
3)   1.632 us    |        _raw_spin_unlock();
3)   1.584 us    |        shrink_dentry_list();
3)   1.776 us    |        task_work_add();
3) + 25.248 us   |      }
3) ! 779.376 us  |    }
3) ! 967.280 us  |  }
```

## 注意事项
在调试过程中，出现追踪指定函数失败的情况：待trace的函数在kernel源码内有定义，但无法设为trace目标对象。

分析是编译时被内联优化，可在源码对应函数加上`noinline`前缀，如：
```c
-static int usb_enumerate_device(struct usb_device *udev) 
+noinline static int usb_enumerate_device(struct usb_device *udev)
```
重新编译后可被正常tracing