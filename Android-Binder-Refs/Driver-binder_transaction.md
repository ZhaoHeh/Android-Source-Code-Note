# binder_transaction 分析

[binder_transaction](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L2424)  
&emsp;[binder_get_node_refs_for_txn](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L2403)  
&emsp;[binder_alloc_new_buf](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L567)  
&emsp;&emsp;[binder_alloc_new_buf_locked](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L378)  
&emsp;&emsp;&emsp;[binder_alloc_buffer_size](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L60)  
&emsp;&emsp;&emsp;[binder_update_page_range](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L181)  
&emsp;[binder_alloc_copy_user_to_buffer](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L1199)  
&emsp;&emsp;[binder_alloc_get_page](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L1140)  

```c++
/*
      struct binder_transaction_data
    |--------------------------------| <-------- &tr（内核空间）
    | tr.target (= ?)                |
    |--------------------------------|
    | tr.cookie  (= ?)               |
    |--------------------------------|
    | tr.code (= ?)                  |
    |--------------------------------|
    | tr.flags (= ?)                 |
    |--------------------------------|
    | tr.sender_pid (= ?)            |
    |--------------------------------|
    | tr.sender_euid (= ?)           |
    |--------------------------------|
    | tr.data_size (= ?)             |
    |--------------------------------|
    | tr.offsets_size (= ?)          |
    |--------------------------------|
    | tr.data.ptr.buffer (= ?)       |
    |--------------------------------|
    | tr.data.ptr.offsets (= ?)      |
    |--------------------------------|
*/
    static void binder_transaction(struct binder_proc *proc,
                    struct binder_thread *thread,
                    struct binder_transaction_data *tr, int reply, // Note: reply是上个函数中的入参cmd == BC_REPLY，由于cmd实际为BC_TRANSACTION，所以reply为0
                    binder_size_t extra_buffers_size)
    {
        int ret;
        struct binder_transaction *t;
        struct binder_work *w;
        struct binder_work *tcomplete;
        binder_size_t buffer_offset = 0;
        binder_size_t off_start_offset, off_end_offset;
        binder_size_t off_min;
        binder_size_t sg_buf_offset, sg_buf_end_offset;
        struct binder_proc *target_proc = NULL;
        struct binder_thread *target_thread = NULL;
        struct binder_node *target_node = NULL;
        struct binder_transaction *in_reply_to = NULL;
        struct binder_transaction_log_entry *e;
        uint32_t return_error = 0;
        uint32_t return_error_param = 0;
        uint32_t return_error_line = 0;
        binder_size_t last_fixup_obj_off = 0;
        binder_size_t last_fixup_min_off = 0;
        struct binder_context *context = proc->context;
        int t_debug_id = atomic_inc_return(&binder_last_id);
        char *secctx = NULL;
        u32 secctx_sz = 0;

        e = binder_transaction_log_add(&binder_transaction_log);
        e->debug_id = t_debug_id;
        e->call_type = reply ? 2 : !!(tr->flags & TF_ONE_WAY);
        e->from_proc = proc->pid;
        e->from_thread = thread->pid;
        e->target_handle = tr->target.handle;
        e->data_size = tr->data_size;
        e->offsets_size = tr->offsets_size;
        strscpy(e->context_name, proc->context->name, BINDERFS_MAX_NAME);

        if (reply) {
            // ... 省略代码
        } else {
// Note: 由于本次代理是ServiceManger的，handle（即上层sServiceManager对应的BinderProxy对应的那个BpBinder对应的handle）值为0，所以走else逻辑
            if (tr->target.handle) {
                // ... 省略代码
            } else {
                mutex_lock(&context->context_mgr_node_lock);
// Note: struct binder_node 类型的 target_node 被赋值
                target_node = context->binder_context_mgr_node;
                if (target_node)
// Note: struct binder_proc 类型的 target_proc 被赋值，对应的就是本次IPC通信的目标进程
                    target_node = binder_get_node_refs_for_txn(
                            target_node, &target_proc,
                            &return_error);
                else
                    return_error = BR_DEAD_REPLY;
                mutex_unlock(&context->context_mgr_node_lock);
                // ... 省略代码
            }
            // ... 省略代码
            binder_inner_proc_lock(proc);
            // ... 省略代码
            if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
                struct binder_transaction *tmp;

                tmp = thread->transaction_stack;
                // ... 省略代码
                while (tmp) {
                    struct binder_thread *from;

                    spin_lock(&tmp->lock);
                    from = tmp->from;
                    if (from && from->proc == target_proc) {
                        atomic_inc(&from->tmp_ref);
                        target_thread = from; // 这里的逻辑没有看懂？？？？？？，target_thread到底是怎么来的？？？？？？
                        spin_unlock(&tmp->lock);
                        break;
                    }
                    spin_unlock(&tmp->lock);
                    tmp = tmp->from_parent;
                }
            }
            binder_inner_proc_unlock(proc);
        }
        // ... 省略代码

        /* TODO: reuse incoming transaction for reply */
// Note: 在内核空间（应该是堆上）分配一个 struct binder_transaction 类型的对象 t，并将所有元素置为0
        t = kzalloc(sizeof(*t), GFP_KERNEL);
        // ... 省略代码
// Note: 在内核空间（应该是堆上）分配一个 struct binder_work 类型的对象 tcomplete，并将所有元素置为0
        tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
        // ... 省略代码
// Note: 非oneway的通信方式，把当前thread保存到 binder_transaction 对象的 from 字段
        if (!reply && !(tr->flags & TF_ONE_WAY))
            t->from = thread;
        else
            t->from = NULL;
        t->sender_euid = task_euid(proc->tsk);
        t->to_proc = target_proc;
        t->to_thread = target_thread;
        t->code = tr->code;
        t->flags = tr->flags;
        t->priority = task_nice(current);
        // ... 省略代码
// Note: Very Very Very Important：binder_alloc_new_buf
// Note: 从目标进程target_proc（本次transaction对应的则是service manager）中分配内存空间，这一步非常重要，因为分配的内存是在目标进程启动时调用mmap映射的内存空间中的，这也是Binder一次拷贝的具体机制
// Note: target_proc->alloc 记录着server端进程启动时执行mmap后映射的内存信息，binder_alloc_new_buf函数的作用就是在映射区域中找到一块合适的内存（用binder_buffer结构表示），用于客户端参数的拷贝；
// Note: 最后，这块server端内核空间和用户空间映射的内存用 binder_transaction.buffer 来指示
        t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
            tr->offsets_size, extra_buffers_size,
            !reply && (t->flags & TF_ONE_WAY), current->tgid);
        // ... 省略代码
        t->buffer->debug_id = t->debug_id;
        t->buffer->transaction = t;
        t->buffer->target_node = target_node;
        t->buffer->clear_on_free = !!(t->flags & TF_CLEAR_BUF);
        // ... 省略代码
        if (binder_alloc_copy_user_to_buffer(
                    &target_proc->alloc,
                    t->buffer, 0,
                    (const void __user *)
                        (uintptr_t)tr->data.ptr.buffer,
                    tr->data_size)) {
// Note: binder_transaction_data.data.ptr.buffer就是名为data的Parcel对象中mData的值(reinterpret_cast<uintptr_t>(mData))，也就是数据的地址
// Note: binder_transaction_data.data_size则是mData对应的数据的大小(mDataSize > mDataPos ? mDataSize : mDataPos)，单位为size_t，拷贝的具体内容如下：
/*
    |----------------------------------|
    |DESCRIPTOR                        |
    |----------------------------------|
    |Context.ACTIVITY_SERVICE          |
    |----------------------------------|
    |----------------------------------|
    |int(allowIsolated true 1)         |
    |----------------------------------|
    |int(DUMP_FLAG_PRIORITY_DEFAULT 8) |
    |----------------------------------|
*/
            // ... 省略代码
            goto err_copy_data_failed;
        }
        if (binder_alloc_copy_user_to_buffer(
                    &target_proc->alloc,
                    t->buffer,
                    ALIGN(tr->data_size, sizeof(void *)),
                    (const void __user *)
                        (uintptr_t)tr->data.ptr.offsets,
                    tr->offsets_size)) {
// Note: binder_transaction_data.data.ptr.offsets就是名为data的Parcel对象中所有flat_binder_object对象的起始地址(reinterpret_cast<uintptr_t>(mObjects))
// Note: binder_transaction_data.offsets_size则是这些flat_binder_object对象总的数据大小(data.ipcObjectsCount()*sizeof(binder_size_t))，拷贝的具体内容如下：
/*
    |----------------------------------|
    |----------------------------------|
    |----------------------------------|
    |flat_binder_object                |
    |----------------------------------|
    |----------------------------------|
    |----------------------------------|
*/
            // ... 省略代码
            goto err_copy_data_failed;
        }
        // ... 省略代码
        off_start_offset = ALIGN(tr->data_size, sizeof(void *));
        buffer_offset = off_start_offset;
        off_end_offset = off_start_offset + tr->offsets_size;
        sg_buf_offset = ALIGN(off_end_offset, sizeof(void *));
        sg_buf_end_offset = sg_buf_offset + extra_buffers_size -
            ALIGN(secctx_sz, sizeof(u64));
        off_min = 0;
        for (buffer_offset = off_start_offset; buffer_offset < off_end_offset;
            buffer_offset += sizeof(binder_size_t)) {
            struct binder_object_header *hdr;
            size_t object_size;
            struct binder_object object;
            binder_size_t object_offset;

            if (binder_alloc_copy_from_buffer(&target_proc->alloc,
                            &object_offset,
                            t->buffer,
                            buffer_offset,
                            sizeof(object_offset))) {
// ... 省略代码
            }
            object_size = binder_get_object(target_proc, t->buffer,
                            object_offset, &object);
// ... 省略代码
            hdr = &object.hdr;
            off_min = object_offset + object_size;
            switch (hdr->type) { // Note: ？？？？？？ hdr->type的具体值是怎么确定的？
            case BINDER_TYPE_BINDER:
            case BINDER_TYPE_WEAK_BINDER:// ... 省略代码
            case BINDER_TYPE_HANDLE:
            case BINDER_TYPE_WEAK_HANDLE: {
                struct flat_binder_object *fp;

                fp = to_flat_binder_object(hdr);
                ret = binder_translate_handle(fp, t, thread);// ... 省略代码
            } break;
            case BINDER_TYPE_FD:// ... 省略代码
            case BINDER_TYPE_FDA:// ... 省略代码
            default:// ... 省略代码
            }
        }
        tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
// 将t->work.type标记为BINDER_WORK_TRANSACTION
        t->work.type = BINDER_WORK_TRANSACTION;

        if (reply) {
// ... 省略代码
// Note: 总的来说，是将tcomplete（BINDER_WORK_TRANSACTION_COMPLETE）写入到当前线程thread
// Note: 将t（BINDER_WORK_TRANSACTION）添加到目标线程target_thread
        } else if (!(t->flags & TF_ONE_WAY)) {// ... 省略代码
            binder_enqueue_deferred_thread_work_ilocked(thread, tcomplete);
            if (!binder_proc_transaction(t, target_proc, target_thread)) // ... 省略代码
        } else {// ... 省略代码
            binder_enqueue_thread_work(thread, tcomplete);
            if (!binder_proc_transaction(t, target_proc, NULL))// ... 省略代码
        }
// ... 省略代码
        return;
// ... 省略代码
    }
```

```c++
    /**
    * binder_get_node_refs_for_txn() - Get required refs on node for txn
    * @node:         struct binder_node for which to get refs
    * @proc:         returns @node->proc if valid
    * @error:        if no @proc then returns BR_DEAD_REPLY
    *
    * User-space normally keeps the node alive when creating a transaction
    * since it has a reference to the target. The local strong ref keeps it
    * alive if the sending process dies before the target process processes
    * the transaction. If the source process is malicious or has a reference
    * counting bug, relying on the local strong ref can fail.
    *
    * Since user-space can cause the local strong ref to go away, we also take
    * a tmpref on the node to ensure it survives while we are constructing
    * the transaction. We also need a tmpref on the proc while we are
    * constructing the transaction, so we take that here as well.
    *
    * Return: The target_node with refs taken or NULL if no @node->proc is NULL.
    * Also sets @proc if valid. If the @node->proc is NULL indicating that the
    * target proc has died, @error is set to BR_DEAD_REPLY
    */
    static struct binder_node *binder_get_node_refs_for_txn(
            struct binder_node *node,
            struct binder_proc **procp,
            uint32_t *error)
    {
        struct binder_node *target_node = NULL;

        binder_node_inner_lock(node);
        if (node->proc) {
            target_node = node;
            binder_inc_node_nilocked(node, 1, 0, NULL);
            binder_inc_node_tmpref_ilocked(node);
            node->proc->tmp_ref++;
            *procp = node->proc;
        } else
            *error = BR_DEAD_REPLY;
        binder_node_inner_unlock(node);

        return target_node;
    }
```

```c++
    /**
    * binder_alloc_new_buf() - Allocate a new binder buffer
    * @alloc:              binder_alloc for this proc
    * @data_size:          size of user data buffer
    * @offsets_size:       user specified buffer offset
    * @extra_buffers_size: size of extra space for meta-data (eg, security context)
    * @is_async:           buffer for async transaction
    * @pid:             pid to attribute allocation to (used for debugging)
    *
    * Allocate a new buffer given the requested sizes. Returns
    * the kernel version of the buffer pointer. The size allocated
    * is the sum of the three given sizes (each rounded up to
    * pointer-sized boundary)
    *
    * Return:   The allocated buffer or %NULL if error
    */
    struct binder_buffer *binder_alloc_new_buf(struct binder_alloc *alloc,
                        size_t data_size,
                        size_t offsets_size,
                        size_t extra_buffers_size,
                        int is_async,
                        int pid)
// Note: struct binder_alloc *alloc     &target_proc->alloc
// Note: size_t data_size               tr->data_size
// Note: size_t offsets_size            tr->offsets_size
// Note: size_t extra_buffers_size      extra_buffers_size
// Note: int is_async                   !reply && (t->flags & TF_ONE_WAY)
// Note: int pid                        current->tgid
    {
        struct binder_buffer *buffer;

        mutex_lock(&alloc->mutex);
        buffer = binder_alloc_new_buf_locked(alloc, data_size, offsets_size,
                            extra_buffers_size, is_async, pid);
        mutex_unlock(&alloc->mutex);
        return buffer;
    }
```

```c++
    static struct binder_buffer *binder_alloc_new_buf_locked(
                    struct binder_alloc *alloc,
                    size_t data_size,
                    size_t offsets_size,
                    size_t extra_buffers_size,
                    int is_async,
                    int pid)
// Note: struct binder_alloc *alloc     &target_proc->alloc
// Note: size_t data_size               tr->data_size
// Note: size_t offsets_size            tr->offsets_size
// Note: size_t extra_buffers_size      extra_buffers_size
// Note: int is_async                   !reply && (t->flags & TF_ONE_WAY)
// Note: int pid                        current->tgid
    {
// Note: *struct binder_alloc*结构包含*struct rb_root*类型的成员free_buffers，*struct rb_root*结构定义如下：
/*
    struct rb_root {
        struct rb_node *rb_node;
    };
*/
// Note: *struct rb_root*结构中包含一个指向根节点的指针
        struct rb_node *n = alloc->free_buffers.rb_node;
        struct binder_buffer *buffer;
        size_t buffer_size;
        struct rb_node *best_fit = NULL;
        void __user *has_page_addr;
        void __user *end_page_addr;
        size_t size, data_offsets_size;
        int ret;
// Note: 先判断server端进程的binder_alloc结构中有没有vma，如果有vma才能正常进行下面的操作
        if (!binder_alloc_get_vma(alloc)) {
            binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
                    "%d: binder_alloc_buf, no vma\n",
                    alloc->pid);
            return ERR_PTR(-ESRCH);
        }
// Note: sizeof(void *))其实就是判断编译目标平台的位宽，一般情况下Android平台都是arm64位的CPU，因此sizeof(void *))结果为8，单位是byte
// Note: ALIGN是一种对齐方式，计算方法很巧妙，我们通过举例说明：ALIGN(0, 8) = 0，ALIGN(1, 8) = 8，ALIGN(5, 8) = 8，ALIGN(8, 8) = 8，ALIGN(10, 8) = 8，应该就能明白了
        data_offsets_size = ALIGN(data_size, sizeof(void *)) +
            ALIGN(offsets_size, sizeof(void *));

        if (data_offsets_size < data_size || data_offsets_size < offsets_size) {
            binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
                    "%d: got transaction with invalid size %zd-%zd\n",
                    alloc->pid, data_size, offsets_size);
            return ERR_PTR(-EINVAL);
        }
        size = data_offsets_size + ALIGN(extra_buffers_size, sizeof(void *));
        if (size < data_offsets_size || size < extra_buffers_size) {
            binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
                    "%d: got transaction with invalid extra_buffers_size %zd\n",
                    alloc->pid, extra_buffers_size);
            return ERR_PTR(-EINVAL);
        }
        if (is_async &&
            alloc->free_async_space < size + sizeof(struct binder_buffer)) {
            binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
                    "%d: binder_alloc_buf size %zd failed, no async space left\n",
                    alloc->pid, size);
            return ERR_PTR(-ENOSPC);
        }

        /* Pad 0-size buffers so they get assigned unique addresses */
        size = max(size, sizeof(void *));
// Note: 到此为止，对tr->data_size、tr->offsets_size、extra_buffers_size做了加工，得到了data_offsets_size和size

        while (n) {
// Note: #define rb_entry(ptr, type, member) container_of(ptr, type, member) 即通过指针变量ptr找到所在容器结构的指针并返回，type是容器结构的类型，member是ptr的类型
// Note: buffer是*struct binder_buffer*类型的指针，由于*struct binder_buffer*中包含了*struct rb_node*结构，所以可以通过rb_node找到对应的binder_buffer
            buffer = rb_entry(n, struct binder_buffer, rb_node);
            BUG_ON(!buffer->free);
            buffer_size = binder_alloc_buffer_size(alloc, buffer);

            if (size < buffer_size) {
                best_fit = n;
                n = n->rb_left;
            } else if (size > buffer_size)
                n = n->rb_right;
            else {
                best_fit = n;
                break;
            }
// Note: while循环是在遍历alloca->free_buffers红黑树，终止条件有三个：
// Note:    (1) 找到最为合适的*struct binder_buffer*对象， *struct binder_buffer*对象管理的内存（buffer_size）和需要的内存（size）正好相等
// Note:    (2) n被置为NULL，这意味着best_fit指示的*struct binder_buffer*对象管理的内存比需要的内存略大，但已经是所有free_buffers中最小的了，已经是红黑树的叶子节点
// Note:    (3) 最不幸的情况，所有alloca->free_buffers管理的内存都比所需要的内存小，best_fit没有被赋值依旧为NULL，已经遍历到管理内存最大的叶子节点了
        }
        if (best_fit == NULL) {
// Note: 这是针对上述情况三处理的，没有啥好办法，只能对现有buffer情况做个统计，然后打印错误日志返回
// Note: 这也就告诉我们这种情况一般不会出现,毕竟binder通信的接口早就在aidl或hidl中定义好了，需要多少*struct binder_buffer*，每个*struct binder_buffer*管理内存的大小都是确定的
            size_t allocated_buffers = 0;
            size_t largest_alloc_size = 0;
            size_t total_alloc_size = 0;
            size_t free_buffers = 0;
            size_t largest_free_size = 0;
            size_t total_free_size = 0;

            for (n = rb_first(&alloc->allocated_buffers); n != NULL;
                n = rb_next(n)) {
                buffer = rb_entry(n, struct binder_buffer, rb_node);
                buffer_size = binder_alloc_buffer_size(alloc, buffer);
                allocated_buffers++;
                total_alloc_size += buffer_size;
                if (buffer_size > largest_alloc_size)
                    largest_alloc_size = buffer_size;
            }
            for (n = rb_first(&alloc->free_buffers); n != NULL;
                n = rb_next(n)) {
                buffer = rb_entry(n, struct binder_buffer, rb_node);
                buffer_size = binder_alloc_buffer_size(alloc, buffer);
                free_buffers++;
                total_free_size += buffer_size;
                if (buffer_size > largest_free_size)
                    largest_free_size = buffer_size;
            }
            binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
                    "%d: binder_alloc_buf size %zd failed, no address space\n",
                    alloc->pid, size);
            binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
                    "allocated: %zd (num: %zd largest: %zd), free: %zd (num: %zd largest: %zd)\n",
                    total_alloc_size, allocated_buffers,
                    largest_alloc_size, total_free_size,
                    free_buffers, largest_free_size);
            return ERR_PTR(-ENOSPC);
        }
        if (n == NULL) {
// Note: 这是针对上述情况二处理的，只需要重新修改一下目标buffer和buffer_size就好
            buffer = rb_entry(best_fit, struct binder_buffer, rb_node);
            buffer_size = binder_alloc_buffer_size(alloc, buffer);
        }
// Note: 情况一也就是最好的情况不需要额外处理，指针变量buffer和管理内存大小buffer_size早在while循环中赋值好了

        binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
                "%d: binder_alloc_buf size %zd got buffer %pK size %zd\n",
                alloc->pid, size, buffer, buffer_size);

        has_page_addr = (void __user *)
            (((uintptr_t)buffer->user_data + buffer_size) & PAGE_MASK);
        WARN_ON(n && buffer_size != size);
        end_page_addr =
            (void __user *)PAGE_ALIGN((uintptr_t)buffer->user_data + size);
        if (end_page_addr > has_page_addr)
            end_page_addr = has_page_addr;
        ret = binder_update_page_range(alloc, 1, (void __user *)
            PAGE_ALIGN((uintptr_t)buffer->user_data), end_page_addr);
// Note: 以上是针对页的操作，暂时还弄不明白？？？？？？
        if (ret)
            return ERR_PTR(ret);

        if (buffer_size != size) {
// Note: 这里还是针对情况做的一个收尾处理，buffer_size != size实际上就是buffer_size略大于size
// Note: binder驱动不浪费，要把多余出来的这点边角料也用一个*struct binder_buffer*表示出来
// Note: *struct binder_buffer*对象用指针变量new_buffer指示
            struct binder_buffer *new_buffer;

            new_buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
            if (!new_buffer) {
                pr_err("%s: %d failed to alloc new buffer struct\n",
                    __func__, alloc->pid);
                goto err_alloc_buf_struct_failed;
            }
            new_buffer->user_data = (u8 __user *)buffer->user_data + size;
// Note: 加入到队列中
            list_add(&new_buffer->entry, &buffer->entry);
            new_buffer->free = 1;
// Note: 加入到红黑树中
            binder_insert_free_buffer(alloc, new_buffer);
        }

        rb_erase(best_fit, &alloc->free_buffers);
        buffer->free = 0;
        buffer->allow_user_free = 0;
        binder_insert_allocated_buffer_locked(alloc, buffer);
// Note: 以上几行操作，从free_buffers红黑树中清除，加入到allocated_buffers红黑树中
        binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
                "%d: binder_alloc_buf size %zd got %pK\n",
                alloc->pid, size, buffer);
        buffer->data_size = data_size;
        buffer->offsets_size = offsets_size;
        buffer->async_transaction = is_async;
        buffer->extra_buffers_size = extra_buffers_size;
        buffer->pid = pid;
        if (is_async) {
            alloc->free_async_space -= size + sizeof(struct binder_buffer);
            binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC_ASYNC,
                    "%d: binder_alloc_buf size %zd async free %zd\n",
                    alloc->pid, size, alloc->free_async_space);
            if (alloc->free_async_space < alloc->buffer_size / 10) {
                /*
                * Start detecting spammers once we have less than 20%
                * of async space left (which is less than 10% of total
                * buffer size).
                */
                debug_low_async_space_locked(alloc, pid);
            }
        }
// Note: 没有错误的话，就返回了找到的buffer
        return buffer;

    err_alloc_buf_struct_failed:
        binder_update_page_range(alloc, 0, (void __user *)
                    PAGE_ALIGN((uintptr_t)buffer->user_data),
                    end_page_addr);
        return ERR_PTR(-ENOMEM);
    }
```

```c++
    static size_t binder_alloc_buffer_size(struct binder_alloc *alloc,
                        struct binder_buffer *buffer)
    {
// Note: *struct binder_alloc*用两种数据结构保存其管理的*struct binder_buffer*类型的数据
// Note:    使用链表保存全部*struct binder_buffer*对象，使用红黑树分别保存两类*struct binder_buffer*对象，allocated和free的
        if (list_is_last(&buffer->entry, &alloc->buffers))
// Note: list_is_last见名知义，是判断当前*struct binder_buffer*对象是否在*struct binder_alloc*所保存的全部*struct binder_buffer*对象列表的末尾
            return alloc->buffer + alloc->buffer_size - buffer->user_data;
// Note: alloc->buffer是base of per-proc address space mapped via mmap
// Note: alloc->buffer_size是size of address space specified via mmap
// Note: buffer->user_data是user pointer to base of buffer space
// Note: 因此本表达式计算出了作为last node的*struct binder_buffer*对象所管理的内存的大小
        return binder_buffer_next(buffer)->user_data - buffer->user_data;
// Note: 因此，本表达式也是为了计算*struct binder_buffer*对象所管理的内存的大小
    }
```

```c++
    static int binder_update_page_range(struct binder_alloc *alloc, int allocate,
                        void __user *start, void __user *end)
    {
        void __user *page_addr;
        unsigned long user_page_addr;
        struct binder_lru_page *page;
        struct vm_area_struct *vma = NULL;
        struct mm_struct *mm = NULL;
        bool need_mm = false;

        binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
                "%d: %s pages %pK-%pK\n", alloc->pid,
                allocate ? "allocate" : "free", start, end);

        if (end <= start)
            return 0;

        trace_binder_update_page_range(alloc, allocate, start, end);

        if (allocate == 0)
            goto free_range;

        for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
            page = &alloc->pages[(page_addr - alloc->buffer) / PAGE_SIZE];
            if (!page->page_ptr) {
                need_mm = true;
                break;
            }
        }

        if (need_mm && mmget_not_zero(alloc->vma_vm_mm))
            mm = alloc->vma_vm_mm;

        if (mm) {
            mmap_read_lock(mm);
            vma = alloc->vma;
        }

        if (!vma && need_mm) {
            binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
                    "%d: binder_alloc_buf failed to map pages in userspace, no vma\n",
                    alloc->pid);
            goto err_no_vma;
        }

        for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
            int ret;
            bool on_lru;
            size_t index;

            index = (page_addr - alloc->buffer) / PAGE_SIZE;
            page = &alloc->pages[index];

            if (page->page_ptr) {
                trace_binder_alloc_lru_start(alloc, index);

                on_lru = list_lru_del(&binder_alloc_lru, &page->lru);
                WARN_ON(!on_lru);

                trace_binder_alloc_lru_end(alloc, index);
                continue;
            }

            if (WARN_ON(!vma))
                goto err_page_ptr_cleared;

            trace_binder_alloc_page_start(alloc, index);
            page->page_ptr = alloc_page(GFP_KERNEL |
                            __GFP_HIGHMEM |
                            __GFP_ZERO);
            if (!page->page_ptr) {
                pr_err("%d: binder_alloc_buf failed for page at %pK\n",
                    alloc->pid, page_addr);
                goto err_alloc_page_failed;
            }
            page->alloc = alloc;
            INIT_LIST_HEAD(&page->lru);

            user_page_addr = (uintptr_t)page_addr;
            ret = vm_insert_page(vma, user_page_addr, page[0].page_ptr);
            if (ret) {
                pr_err("%d: binder_alloc_buf failed to map page at %lx in userspace\n",
                    alloc->pid, user_page_addr);
                goto err_vm_insert_page_failed;
            }

            if (index + 1 > alloc->pages_high)
                alloc->pages_high = index + 1;

            trace_binder_alloc_page_end(alloc, index);
        }
        if (mm) {
            mmap_read_unlock(mm);
            mmput(mm);
        }
        return 0;

    free_range:
        for (page_addr = end - PAGE_SIZE; 1; page_addr -= PAGE_SIZE) {
            bool ret;
            size_t index;

            index = (page_addr - alloc->buffer) / PAGE_SIZE;
            page = &alloc->pages[index];

            trace_binder_free_lru_start(alloc, index);

            ret = list_lru_add(&binder_alloc_lru, &page->lru);
            WARN_ON(!ret);

            trace_binder_free_lru_end(alloc, index);
            if (page_addr == start)
                break;
            continue;

    err_vm_insert_page_failed:
            __free_page(page->page_ptr);
            page->page_ptr = NULL;
    err_alloc_page_failed:
    err_page_ptr_cleared:
            if (page_addr == start)
                break;
        }
    err_no_vma:
        if (mm) {
            mmap_read_unlock(mm);
            mmput(mm);
        }
        return vma ? -ENOMEM : -ESRCH;
    }
```

```c++
    /**
    * binder_alloc_copy_user_to_buffer() - copy src user to tgt user
    * @alloc: binder_alloc for this proc
    * @buffer: binder buffer to be accessed
    * @buffer_offset: offset into @buffer data
    * @from: userspace pointer to source buffer
    * @bytes: bytes to copy
    *
    * Copy bytes from source userspace to target buffer.
    *
    * Return: bytes remaining to be copied
    */
    unsigned long
    binder_alloc_copy_user_to_buffer(struct binder_alloc *alloc,
                    struct binder_buffer *buffer,
                    binder_size_t buffer_offset,
                    const void __user *from,
                    size_t bytes)
    {
        if (!check_buffer(alloc, buffer, buffer_offset, bytes))
            return bytes;

        while (bytes) {
            unsigned long size;
            unsigned long ret;
            struct page *page;
            pgoff_t pgoff;
            void *kptr;

            page = binder_alloc_get_page(alloc, buffer,
                            buffer_offset, &pgoff);
            size = min_t(size_t, bytes, PAGE_SIZE - pgoff);
            kptr = kmap(page) + pgoff;
// Note: 现在，终于把IServiceManager.Stub.Proxy.addService的入参从用户空间拷贝到了内核空间
            ret = copy_from_user(kptr, from, size);
            kunmap(page);
            if (ret)
                return bytes - size + ret;
            bytes -= size;
            from += size;
            buffer_offset += size;
        }
        return 0;
    }
```

```c++
    /**
    * binder_alloc_get_page() - get kernel pointer for given buffer offset
    * @alloc: binder_alloc for this proc
    * @buffer: binder buffer to be accessed
    * @buffer_offset: offset into @buffer data
    * @pgoffp: address to copy final page offset to
    *
    * Lookup the struct page corresponding to the address
    * at @buffer_offset into @buffer->user_data. If @pgoffp is not
    * NULL, the byte-offset into the page is written there.
    *
    * The caller is responsible to ensure that the offset points
    * to a valid address within the @buffer and that @buffer is
    * not freeable by the user. Since it can't be freed, we are
    * guaranteed that the corresponding elements of @alloc->pages[]
    * cannot change.
    *
    * Return: struct page
    */
    static struct page *binder_alloc_get_page(struct binder_alloc *alloc,
                        struct binder_buffer *buffer,
                        binder_size_t buffer_offset,
                        pgoff_t *pgoffp)
    {
        binder_size_t buffer_space_offset = buffer_offset +
            (buffer->user_data - alloc->buffer);
        pgoff_t pgoff = buffer_space_offset & ~PAGE_MASK;
        size_t index = buffer_space_offset >> PAGE_SHIFT;
        struct binder_lru_page *lru_page;

        lru_page = &alloc->pages[index];
        *pgoffp = pgoff;
        return lru_page->page_ptr;
    }
```
