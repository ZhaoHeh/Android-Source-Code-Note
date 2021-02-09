# 其他一些说明

## ActivityManagerService对应的BBinder是怎么来的

*com.android.server.am.ActivityManagerService*的继承关系如下：  

[ActivityManagerService][ActivityManagerServiceLink] -> [android.os.IServiceManager.Stub][IServiceManagerStubLink] -> [android.os.Binder][BinderLink]

[ActivityManagerServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=417
[IServiceManagerStubLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=111
[BinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Binder.java;l=584

ActivityManagerService的实例化必然伴随着超类Binder的实例化，Binder的构造器如下：  

[Binder#Binder][BinderCnstctrLink]  
&emsp;[Binder#getNativeBBinderHolder][getNativeBBinderHolderLink]  

[BinderCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Binder.java;l=600
[getNativeBBinderHolderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=1021

```java
    public Binder(@Nullable String descriptor)  {
        mObject = getNativeBBinderHolder();
        NoImagePreloadHolder.sRegistry.registerNativeAllocation(this, mObject);

        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Binder> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Binder class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        mDescriptor = descriptor;
    }
// 代码路径：frameworks/base/core/java/android/os/Binder.java
```

native的holder对象实例化之后，会在[适当的时机](./Android-Binder-2.md)实例化其中的成员变量mBinder所指向的JavaBBinder：  

[JavaBBinderHolder::get][getLink]  
&emsp;[JavaBBinder::JavaBBinder][JavaBBinderCnstctrLink]  
&emsp;&emsp;[BBinder::BBinder][BBinderCnstctrLink]  

[getLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=462
[JavaBBinderCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=343
[BBinderCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Binder.cpp;l=148

```c++
    sp<JavaBBinder> get(JNIEnv* env, jobject obj)
    {
        AutoMutex _l(mLock);
        sp<JavaBBinder> b = mBinder.promote();
        if (b == NULL) {
            b = new JavaBBinder(env, obj);
            if (mVintf) {
                ::android::internal::Stability::markVintf(b.get());
            }
            if (mExtension != nullptr) {
                b.get()->setExtension(mExtension);
            }
            mBinder = b;
            ALOGV("Creating JavaBinder %p (refs %p) for Object %p, weakCount=%" PRId32 "\n",
                 b.get(), b->getWeakRefs(), obj, b->getWeakRefs()->getWeakCount());
        }

        return b;
    }
// 代码路径：frameworks/base/core/jni/android_util_Binder.cpp
```

最终，我们得到的类关系图如下：  

Binder  
&emsp;&emsp;&emsp;|  
&emsp;&emsp;&emsp;|mObject  
&emsp;&emsp;&emsp;|  
JavaBBinderHolder  
&emsp;&emsp;&emsp;|  
&emsp;&emsp;&emsp;|mBinder  
&emsp;&emsp;&emsp;|  
JavaBBinder -> BBinder  

## Pracel类的有关说明

[Parcel#obtain][ParcelObtainLink]  
&emsp;[Parcel#Parcel][ParcelParcelLink]  
&emsp;&emsp;[Parcel#init][ParcelinitLink]  
&emsp;&emsp;&emsp;[Parcel#nativeCreate][ParcelnativeCreateLink]  
&emsp;&emsp;&emsp;&emsp;[ParcelRef::create][ParcelRefCreateLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[ParcelRef::ParcelRef][ParcelRefCnstctrLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::Parcel][ParcelCnstctrLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::initState][ParcelInitStateLink]  

```c++
void Parcel::initState()
{
    LOG_ALLOC("Parcel %p: initState", this);
    mError = NO_ERROR;
    mData = nullptr;  // Note: uint8_t* mData; Parcel中包装的所有数据所在内存的首地址
    mDataSize = 0;    // Note: size_t mDataSize; Parcel当前已有数据的大小，单位为size_t
    mDataCapacity = 0;// Note: size_t mDataCapacity; Parcel中当前数据的容量，单位为size_t
    mDataPos = 0;     // Note: mutable size_t mDataPos; Parcel中已有数据相对于首地址的偏移量，(mData+mDataPos)也就是将要添加的新数据的首地址
    ALOGV("initState Setting data size of %p to %zu", this, mDataSize);
    ALOGV("initState Setting data pos of %p to %zu", this, mDataPos);
    mObjects = nullptr;
    mObjectsSize = 0;
    mObjectsCapacity = 0;
    mNextObjectHint = 0;
    mObjectsSorted = false;
    mHasFds = false;
// ... 省略代码
}
// 代码路径：frameworks/native/libs/binder/Parcel.cpp
```

[ParcelObtainLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=424
[ParcelParcelLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=3510
[ParcelinitLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=3518
[ParcelnativeCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_os_Parcel.cpp;l=531
[ParcelRefCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/include/binder/ParcelRef.h;l=33
[ParcelRefCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/include/binder/ParcelRef.h;l=38
[ParcelCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=268
[ParcelInitStateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=2902

到此为止，Parcel类中没有任何数据，mData，mDataSize，mDataCapacity，mDataPos是0/空。  

[Parcel#writeInterfaceToken][writeInterfaceTokenLink]  
&emsp;[Parcel#nativeWriteInterfaceToken][nativeWriteInterfaceTokenLink]  
&emsp;&emsp;[Parcel::writeInterfaceToken][nParcelWriteInterfaceTokenLink]  
&emsp;&emsp;&emsp;[Parcel::writeInt32][nParcelWriteInt32Link]  
&emsp;&emsp;&emsp;&emsp;[Parcel::writeAligned][nParcelWriteAlignedLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::growData][nParcelGrowDataLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::continueWrite][nParcelContinueWriteLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::finishWrite][nParcelfinishWriteLink]  

```c++
template<class T>
status_t Parcel::writeAligned(T val) {
    static_assert(PAD_SIZE_UNSAFE(sizeof(T)) == sizeof(T));

    if ((mDataPos+sizeof(val)) <= mDataCapacity) {
restart_write:
        *reinterpret_cast<T*>(mData+mDataPos) = val;
// Note: *(reinterpret_cast<T*>(mData+mDataPos)) = val;
        return finishWrite(sizeof(val));
    }

    status_t err = growData(sizeof(val));
    if (err == NO_ERROR) goto restart_write;
    return err;
}

status_t Parcel::growData(size_t len)
{
    if (len > INT32_MAX) {
        // don't accept size_t values which may have come from an
        // inadvertent conversion from a negative int.
        return BAD_VALUE;
    }

    if (len > SIZE_MAX - mDataSize) return NO_MEMORY; // overflow
    if (mDataSize + len > SIZE_MAX / 3) return NO_MEMORY; // overflow
    size_t newSize = ((mDataSize+len)*3)/2;
    return (newSize <= mDataSize)
            ? (status_t) NO_MEMORY
            : continueWrite(std::max(newSize, (size_t) 128));
}

status_t Parcel::continueWrite(size_t desired)
{
    if (desired > INT32_MAX) {
// ... 省略代码
    }

    // If shrinking, first adjust for any objects that appear
    // after the new data size.
    size_t objectsSize = mObjectsSize;
    if (desired < mDataSize) {
// ... 省略代码
    }

    if (mOwner) {
// ... 省略代码
    } else if (mData) {
// ... 省略代码
    } else {
        // This is the first data.  Easy!
        uint8_t* data = (uint8_t*)malloc(desired);
        if (!data) {
// ... 省略代码
        }

        if(!(mDataCapacity == 0 && mObjects == nullptr
// ... 省略代码
        }

        LOG_ALLOC("Parcel %p: allocating with %zu capacity", this, desired);
        gParcelGlobalAllocSize += desired;
        gParcelGlobalAllocCount++;

        mData = data;
        mDataSize = mDataPos = 0;
        ALOGV("continueWrite Setting data size of %p to %zu", this, mDataSize);
        ALOGV("continueWrite Setting data pos of %p to %zu", this, mDataPos);
        mDataCapacity = desired;
    }

    return NO_ERROR;
}
// 代码路径：frameworks/native/libs/binder/Parcel.cpp
```

[writeInterfaceTokenLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=656
[nativeWriteInterfaceTokenLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_os_Parcel.cpp;l=688
[nParcelWriteInterfaceTokenLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=535
[nParcelWriteInt32Link]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=967
[nParcelWriteAlignedLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=1594
[nParcelGrowDataLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=2657
[nParcelContinueWriteLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=2740
[nParcelfinishWriteLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=642

## 每一个Zygote孵化的进程，都已经自动开启了Binder线程池

```c++
    virtual void onStarted()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();

        AndroidRuntime* ar = AndroidRuntime::getRuntime();
        ar->callMain(mClassName, mClass, mArgs);

        IPCThreadState::self()->stopProcess();
        hardware::IPCThreadState::self()->stopProcess();
    }
// 代码路径：frameworks/base/cmds/app_process/app_main.cpp
```

[AppRuntime::onStarted](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/cmds/app_process/app_main.cpp;l=79)

## 驱动中的数据结构

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
        struct binder_proc *proc = filp->private_data;

        if (proc->tsk != current->group_leader)
            return -EINVAL;
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
