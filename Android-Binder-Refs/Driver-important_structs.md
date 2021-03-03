# 驱动中的数据结构

```c++
    struct binder_proc {
        struct hlist_node proc_node;
        struct rb_root threads;
        struct rb_root nodes;
        struct rb_root refs_by_desc;
        struct rb_root refs_by_node;
        struct list_head waiting_threads;
        int pid;
        struct task_struct *tsk;
        struct hlist_node deferred_work_node;
        int deferred_work;
        bool is_dead;

        struct list_head todo;
        struct binder_stats stats;
        struct list_head delivered_death;
        int max_threads;
        int requested_threads;
        int requested_threads_started;
        int tmp_ref;
        long default_priority;
        struct dentry *debugfs_entry;
        struct binder_alloc alloc;
        struct binder_context *context;
        spinlock_t inner_lock;
        spinlock_t outer_lock;
        struct dentry *binderfs_entry;
    };
```

## 驱动和用户态都用到的数据结构

```c++
struct binder_write_read {
  binder_size_t write_size;
  binder_size_t write_consumed;
  binder_uintptr_t write_buffer;
  binder_size_t read_size;
  binder_size_t read_consumed;
  binder_uintptr_t read_buffer;
};
// 代码路径：/bionic/libc/kernel/uapi/linux/android/binder.h
```

[binder_write_read](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/kernel/uapi/linux/android/binder.h;l=90)  
