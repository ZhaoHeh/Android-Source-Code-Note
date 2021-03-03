# binder驱动的mmap发生了什么

native层的[android::ProcessState][ProcessStateLink]和驱动中的[struct binder_proc][struct_binder_proc_lk]结构对应，在任意一个ProcessState对象实例化的时候，必然会打开Binder驱动，此时**struct binder_proc**结构就会被allocate。  

[ProcessState::self][ProcessStateSelfLink]  
&emsp;[ProcessState::init][ProcessStateInitLink]  
&emsp;&emsp;[ProcessState::ProcessState][ProcessStateCntrctLink]  
&emsp;&emsp;&emsp;[open_driver][open_driver_lk]  
&emsp;&emsp;&emsp;&emsp;[open(driver, O_RDWR | O_CLOEXEC)][open_systemCall_lk]  
&emsp;&emsp;&emsp;&emsp;[ioctl(fd, BINDER_VERSION, &vers)][ioclt_BINDER_VERSION_lk]  
&emsp;&emsp;&emsp;[mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0)][mmap_systemCall_lk]  

[ProcessStateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=75
[struct_binder_proc_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L459

[ProcessStateSelfLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=75
[ProcessStateInitLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=90
[ProcessStateCntrctLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=395
[open_driver_lk]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=367
[open_systemCall_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L5194
[ioclt_BINDER_VERSION_lk]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=372
[mmap_systemCall_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L5167

```c++
    const struct file_operations binder_fops = {
        .owner = THIS_MODULE,
        .poll = binder_poll,
        .unlocked_ioctl = binder_ioctl,
        .compat_ioctl = compat_ptr_ioctl,
        .mmap = binder_mmap,
        .open = binder_open,
        .flush = binder_flush,
        .release = binder_release,
    };

    static int binder_open(struct inode *nodp, struct file *filp)
    {
        struct binder_proc *proc, *itr;
// ... 省略代码
        proc = kzalloc(sizeof(*proc), GFP_KERNEL);
// ... 省略代码
        proc->tsk = current->group_leader;
        INIT_LIST_HEAD(&proc->todo);
        proc->default_priority = task_nice(current);
// ... 省略代码
        proc->context = &binder_dev->context;
        binder_alloc_init(&proc->alloc);

        binder_stats_created(BINDER_STAT_PROC);
        proc->pid = current->group_leader->pid;
        INIT_LIST_HEAD(&proc->delivered_death);
        INIT_LIST_HEAD(&proc->waiting_threads);
        filp->private_data = proc;
// ... 省略代码
        hlist_add_head(&proc->proc_node, &binder_procs);
        mutex_unlock(&binder_procs_lock);

// ... 省略代码
        return 0;
    }

    static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
    {
// Note: 关于虚存管理的最基本的管理单元应该是struct vm_area_struct了，它描述的是一段连续的、具有相同访问属性的虚存空间，该虚存空间的大小为物理内存页面的整数倍
        struct binder_proc *proc = filp->private_data;

        // ... 省略代码
        vma->vm_flags |= VM_DONTCOPY | VM_MIXEDMAP;
        vma->vm_flags &= ~VM_MAYWRITE;

        vma->vm_ops = &binder_vm_ops;
        vma->vm_private_data = proc;

        return binder_alloc_mmap_handler(&proc->alloc, vma);
    }
```

```c++
    /**
    * binder_alloc_mmap_handler() - map virtual address space for proc
    * @alloc: alloc structure for this proc
    * @vma: vma passed to mmap()
    *
    * Called by binder_mmap() to initialize the space specified in
    * vma for allocating binder buffers
    *
    * Return:
    *      0 = success
    *      -EBUSY = address space already mapped
    *      -ENOMEM = failed to map memory to given address space
    */
    int binder_alloc_mmap_handler(struct binder_alloc *alloc,
                    struct vm_area_struct *vma)
    {
        int ret;
        const char *failure_string;
        struct binder_buffer *buffer;

        mutex_lock(&binder_alloc_mmap_lock);
        if (alloc->buffer_size) {
            ret = -EBUSY;
            failure_string = "already mapped";
            goto err_already_mapped;
        }
        alloc->buffer_size = min_t(unsigned long, vma->vm_end - vma->vm_start,
                    SZ_4M);
        mutex_unlock(&binder_alloc_mmap_lock);

        alloc->buffer = (void __user *)vma->vm_start;

        alloc->pages = kcalloc(alloc->buffer_size / PAGE_SIZE,
                    sizeof(alloc->pages[0]),
                    GFP_KERNEL);
        if (alloc->pages == NULL) {
            ret = -ENOMEM;
            failure_string = "alloc page array";
            goto err_alloc_pages_failed;
        }

        buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
        if (!buffer) {
            ret = -ENOMEM;
            failure_string = "alloc buffer struct";
            goto err_alloc_buf_struct_failed;
        }

        buffer->user_data = alloc->buffer;
        list_add(&buffer->entry, &alloc->buffers);
        buffer->free = 1;
        binder_insert_free_buffer(alloc, buffer);
        alloc->free_async_space = alloc->buffer_size / 2;
        binder_alloc_set_vma(alloc, vma);
        mmgrab(alloc->vma_vm_mm);

        return 0;

    err_alloc_buf_struct_failed:
        kfree(alloc->pages);
        alloc->pages = NULL;
    err_alloc_pages_failed:
        alloc->buffer = NULL;
        mutex_lock(&binder_alloc_mmap_lock);
        alloc->buffer_size = 0;
    err_already_mapped:
        mutex_unlock(&binder_alloc_mmap_lock);
        binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
                "%s: %d %lx-%lx %s failed %d\n", __func__,
                alloc->pid, vma->vm_start, vma->vm_end,
                failure_string, ret);
        return ret;
    }
```

[binder_mmap][binder_mmap_lk]  
&emsp;[binder_alloc_mmap_handler][binder_alloc_mmap_handler_lk]  
&emsp;&emsp;[binder_insert_free_buffer][binder_insert_free_buffer_lk]  

[binder_mmap_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L4780
[binder_alloc_mmap_handler_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L741
[binder_insert_free_buffer_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L68

[binder_alloc_new_buf][binder_alloc_new_buf_lk]  
&emsp;[binder_alloc_new_buf_locked][binder_alloc_new_buf_locked_lk]  

[binder_alloc_new_buf_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L567
[binder_alloc_new_buf_locked_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L378
