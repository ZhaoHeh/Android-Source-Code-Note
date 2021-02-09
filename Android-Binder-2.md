# 一、System Service（server端）注册service的过程

TODO：驱动中的filp参数？  
TODO：mIn是不是映射到内核空间，binder驱动管理的地址？  

## AMS（ActivityManagerService）注册过程分析

[ActivityManagerService#setSystemProcess][AMSsetSystemProcessLink]  
&emsp;[ServiceManager#addService][SvcMgraddServiceLink]  
&emsp;&emsp;[ServiceManagerProxy#addService][SMPaddServiceLink]  
&emsp;&emsp;&emsp;[IServiceManager.Stub.Proxy#addService][ISMSPaddServiceLink]  

[AMSsetSystemProcessLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=2102
[SvcMgraddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=179
[SMPaddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=70
[ISMSPaddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=414

在第一篇已经分析过ServiceManager的实际组成，这里不在赘述

### 1. 第一次封装：Stub类型的system service作为一个参数的加工过程

&emsp;&emsp;&emsp;&emsp;[Parcel#writeStrongBinder][ParcelwriteStrongBinderLink]（这里IBinder类型的参数，实际类型为ActivityManagerService，父类型为IActivityManager.Stub）  
&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel#nativeWriteStrongBinder][nativeWriteStrongBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[ibinderForJavaObject][ibinderForJavaObjectLink]（通过传进来的Java对象ActivityManagerService, 找到其对应的BBinder对象）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[JavaBBinderHolder::get][JavaBBinderHoldergetLink]（这里返回的是一个JavaBBinder对象，继承自BBinder）  
为什么ActivityManagerService对象会有一个对应的BBinder对象，请参考[这里](./Android-Binder-6.md)。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::writeStrongBinder][CppParcelWriteStrongBinderLink]（Parcel的构造和关键成员变量参考[这里](./Android-Binder-6.md)）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::flattenBinder][ParcelFlattenBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[BBinder::localBinder][BBinderLocalBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::writeObject][CppParcelWriteObjectLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::finishWrite][CppParcelFinishWriteLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::finishFlattenBinder][CppParcelFinishFlattenBinderLink]  

[ParcelwriteStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=844
[nativeWriteStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_os_Parcel.cpp;l=327
[ibinderForJavaObjectLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=783
[JavaBBinderHoldergetLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=458
[CppParcelWriteStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=1136
[ParcelFlattenBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=192
[BBinderLocalBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Binder.cpp;l=255
[CppParcelWriteObjectLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=1376
[CppParcelFinishWriteLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=642
[CppParcelFinishFlattenBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=167

```c++
sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;

    // Instance of Binder?
    if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env->GetLongField(obj, gBinderOffsets.mObject);
        return jbh->get(env, obj);
    }

    // Instance of BinderProxy?
    if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        return getBPNativeData(env, obj)->mObject;
    }

    ALOGW("ibinderForJavaObject: %p is not a Binder object", obj);
    return NULL;
}
// 代码路径：frameworks/base/core/jni/android_util_Binder.cpp

status_t Parcel::flattenBinder(const sp<IBinder>& binder)
{
    flat_binder_object obj;
    obj.flags = FLAT_BINDER_FLAG_ACCEPTS_FDS;
// ... 省略代码
    if (binder != nullptr) {
        BBinder *local = binder->localBinder();
        if (!local) {
// ... 省略代码
        } else {
// ... 省略代码
            obj.hdr.type = BINDER_TYPE_BINDER;
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    } else {
// ... 省略代码
    }

    obj.flags |= schedBits;

    status_t status = writeObject(obj, false);
    if (status != OK) return status;

    return finishFlattenBinder(binder);
}
// 代码路径：frameworks/native/libs/binder/Parcel.cpp

struct flat_binder_object {
  struct binder_object_header hdr;
  __u32 flags;
  union {
// Note: union，在一个“union”内可以定义多种不同的数据类型， 一个被说明为该“union”类型的变量中，允许装入该“union”所定义的任何一种数据.
// Note: 这些数据共享同一段内存，以达到节省空间的目的，union变量所占用的内存长度等于最长的成员的内存长度。
    binder_uintptr_t binder;
    __u32 handle;
  };
  binder_uintptr_t cookie;
};
// 代码路径：bionic/libc/kernel/uapi/linux/android/binder.h
```

总结：  
Parcel类将AMS对应的BBinder对象封装进Binder驱动能读懂的**flat_binder_object**结构中，再将flat_binder_object对象写入Parcel::mData指向的数据包里。  
而对于整个addService函数而言，所有参数都被封装进了Parcel类型的data对象中。  

### 2. BpBinder开始transact：第二次封装

&emsp;&emsp;&emsp;&emsp;[BinderProxy#transact][BinderProxytransactLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[**native BinderProxy#transactNative**][BinderProxytransactNativeLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[BpBinder::transact][BpBindertransactLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[IPCThreadState::transact][IPCThreadStatetransactLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[IPCThreadState::writeTransactionData][writeTransactionDataLink]（打包要发送给binder驱动的数据）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::write][nParcelWriteLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::writeInplace][nParcelwriteInplaceLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[memcpy]  

[BinderProxytransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/BinderProxy.java;l=495
[BinderProxytransactNativeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=1377
[BpBindertransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp;l=213
[IPCThreadStatetransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=688
[writeTransactionDataLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=1078
[nParcelWriteLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=686
[nParcelwriteInplaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=702

```c++
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
// Note: 对参数做一些说明
// Note: cmd            BC_TRANSACTION enum类型 IPCThreadState::transact中赋值
// Note: binderFlags    32位int类型 最初是0 加工过程中的赋值有: flags |= FLAG_COLLECT_NOTED_APP_OPS, flags = flags & ~FLAG_PRIVATE_VENDOR, flags |= TF_ACCEPT_FDS
// Note: handle         const int32_t BpBinder::mHandle, service manager对应的handle实际值就是0
// Note: code           IServiceManager.Stub.Proxy#addService中赋值，此处为IServiceManager.Stub.TRANSACTION_addService, 实际值为3
// Note: data           native Parcel类的引用类型，封装了IServiceManager.Stub.Proxy#addService的输入参数
// Note: statusBuffer   nullptr
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
// ... 省略代码
    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    }
// ... 省略代码
    mOut.writeInt32(cmd); // cmd的实际值为BC_TRANSACTION
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
// 代码路径：frameworks/native/libs/binder/IPCThreadState.cpp
```

对将要发送到Binder的数据进一步封装成**binder_transaction_data**结构类型，再将binder_transaction_data中的数据写入IPCThreadState::mOut中。  
说明：  
IPCThreadState::mOut：当前线程和Binder驱动通信时输入到Binder驱动中的数据；  
IPCThreadState::mIn：当前线程和Binder驱动通信时从Binder驱动中读出的数据。  

### 3 开始和驱动通信：对数据第三次封装

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[IPCThreadState::waitForResponse][waitForResponseLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[IPCThreadState::talkWithDriver][talkWithDriverLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)][IOCTLSystemCallDriverLink]  

```c++
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        // ... 省略代码
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();
// ... 省略代码
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:// ... 省略代码
        case BR_DEAD_REPLY:// ... 省略代码
        case BR_FAILED_REPLY:// ... 省略代码
        case BR_FROZEN_REPLY:// ... 省略代码
        case BR_ACQUIRE_RESULT:// ... 省略代码
        case BR_REPLY:// ... 省略代码
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}

status_t IPCThreadState::talkWithDriver(bool doReceive) // Note: doReceive 默认为true
{
// ... 省略代码
    binder_write_read bwr;

    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
// Note: 将IPCThreadState::mOut的数据地址封装进binder_write_read结构中
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
// Note: 将IPCThreadState::mIn的数据地址封装进binder_write_read结构中
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

// ... 省略代码

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
// ... 省略代码
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
// ... 省略代码
    } while (err == -EINTR);
// ... 省略代码
    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
// ... 省略代码
            else {
                mOut.setDataSize(0);
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
// ... 省略代码
        }
        return NO_ERROR;
    }

    return err;
}
// 代码路径：frameworks/native/libs/binder/IPCThreadState.cpp
```

[waitForResponseLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=870
[talkWithDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=965
[IOCTLSystemCallDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=1018

在IPCThreadState::talkWithDriver中做了第三次也是最后一次封装，将IPCThreadState::mOut和IPCThreadState::mIn封装到**binder_write_read**结构中；  
将binder_write_read结构在用户空间的虚拟地址通过ioctl传递到驱动中，数据最终以**binder_write_read**的格式由客户端与驱动交互。  

### 4. 下面进入驱动执行

[binder_ioctl][binder_ioctl_lk]  

```c++
    static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
// Note: 参数说明
// Note: filp   文件指针？？？？？？
// Note: cmd    BINDER_WRITE_READ
// Note: arg    用户空间中binder_write_read结构对象的地址
    {
        int ret;
// Note: 获得该进程在Binder驱动中对应的结构体
        struct binder_proc *proc = filp->private_data;
        struct binder_thread *thread;
        unsigned int size = _IOC_SIZE(cmd);
// Note: ubuf是一个指针变量，值等于unsigned long类型的arg，arg就是ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)中的&bwr，也就是用户空间被bwr结构的指针，所以ubuf的值就等于用户空间bwr结构的虚拟地址。
        void __user *ubuf = (void __user *)arg;

        /*pr_info("binder_ioctl: %d:%d %x %lx\n",
                proc->pid, current->pid, cmd, arg);*/

        binder_selftest_alloc(&proc->alloc);

        trace_binder_ioctl(cmd, arg);
// Note: wait_event_interruptible 会令当前线程进入休眠状态，但前提是binder_stop_on_user_error < 2为false.
        ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
        if (ret)
            goto err_unlocked;
// Note: 查找或创建当前线程对应的**struct binder_thread**结构
        thread = binder_get_thread(proc);
// ... 省略代码
        switch (cmd) {
        case BINDER_WRITE_READ:
            ret = binder_ioctl_write_read(filp, cmd, arg, thread);
            if (ret)
                goto err;
            break;
        case BINDER_SET_MAX_THREADS:// ... 省略代码
        case BINDER_SET_CONTEXT_MGR_EXT:// ... 省略代码
        case BINDER_SET_CONTEXT_MGR:// ... 省略代码
        case BINDER_THREAD_EXIT:// ... 省略代码
        case BINDER_VERSION:// ... 省略代码
        case BINDER_GET_NODE_INFO_FOR_REF:// ... 省略代码
        case BINDER_GET_NODE_DEBUG_INFO:// ... 省略代码
        default:
            ret = -EINVAL;
            goto err;
        }
        ret = 0;
    err:
        if (thread)
            thread->looper_need_return = false;
        wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
        if (ret && ret != -ERESTARTSYS)
            pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
    err_unlocked:
        trace_binder_ioctl_done(ret);
        return ret;
    }
```

&emsp;[binder_ioctl_write_read][binder_ioctl_write_read_lk]  

```c++
    static int binder_ioctl_write_read(struct file *filp,
                    unsigned int cmd, unsigned long arg,
                    struct binder_thread *thread)
    {
        int ret = 0;
        struct binder_proc *proc = filp->private_data;
        unsigned int size = _IOC_SIZE(cmd);
        void __user *ubuf = (void __user *)arg;
        struct binder_write_read bwr;
// ... 省略代码
// Note: 从用户空间ubuf指示的位置，拷贝sizeof(bwr)大小的数据到内核空间&bwr指示的位置，也就是传说中Binder一次拷贝的发生处
        if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
            ret = -EFAULT;
            goto out;
        }
// ... 省略代码
        if (bwr.write_size > 0) {
            ret = binder_thread_write(proc, thread,
// Note: bwr.write_buffer对应的就是IPCThreadState::mOut的数据地址，也就是Parcel::mData指针变量所指的地址
// Note: bwr.write_size就是IPCThreadState::mOut中数据的大小
                        bwr.write_buffer,
                        bwr.write_size,
                        &bwr.write_consumed);
// ... 省略代码
        }
        if (bwr.read_size > 0) {
            ret = binder_thread_read(proc, thread, bwr.read_buffer,
                        bwr.read_size,
                        &bwr.read_consumed,
                        filp->f_flags & O_NONBLOCK);
            trace_binder_read_done(ret);
            binder_inner_proc_lock(proc);
            if (!binder_worklist_empty_ilocked(&proc->todo))
                binder_wakeup_proc_ilocked(proc);
            binder_inner_proc_unlock(proc);
            if (ret < 0) {
                if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                    ret = -EFAULT;
                goto out;
            }
        }
        binder_debug(BINDER_DEBUG_READ_WRITE,
                "%d:%d wrote %lld of %lld, read return %lld of %lld\n",
                proc->pid, thread->pid,
                (u64)bwr.write_consumed, (u64)bwr.write_size,
                (u64)bwr.read_consumed, (u64)bwr.read_size);
        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
            ret = -EFAULT;
            goto out;
        }
    out:
        return ret;
    }
```

&emsp;&emsp;[binder_thread_write][binder_thread_write_lk]  

```c++
    static int binder_thread_write(struct binder_proc *proc,
                struct binder_thread *thread,
                binder_uintptr_t binder_buffer, size_t size,
                binder_size_t *consumed)
    {
        uint32_t cmd;
        struct binder_context *context = proc->context;

        void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
        void __user *ptr = buffer + *consumed;
        void __user *end = buffer + size;

        while (ptr < end && thread->return_error.cmd == BR_OK) {
            int ret;
// Note: 下面开始拷贝用户空间的cmd，此时应该是之前装入的BC_TRANSACTION
            if (get_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            trace_binder_command(cmd);
            if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {
                atomic_inc(&binder_stats.bc[_IOC_NR(cmd)]);
                atomic_inc(&proc->stats.bc[_IOC_NR(cmd)]);
                atomic_inc(&thread->stats.bc[_IOC_NR(cmd)]);
            }
            switch (cmd) {
            case BC_INCREFS:
            case BC_ACQUIRE:
            case BC_RELEASE:
            case BC_DECREFS:
            case BC_INCREFS_DONE:// ... 省略代码
            case BC_ACQUIRE_DONE:// ... 省略代码
            case BC_ATTEMPT_ACQUIRE:// ... 省略代码
            case BC_ACQUIRE_RESULT:// ... 省略代码
            case BC_FREE_BUFFER:// ... 省略代码
            case BC_TRANSACTION_SG:
            case BC_REPLY_SG:// ... 省略代码
            case BC_TRANSACTION:
            case BC_REPLY: {
                struct binder_transaction_data tr;
// Note: 首先这里要复习一下数据在用户态的封装: 
// Note:    Parcel类型的data封装了addService的入参，data又被封装进binder_transaction_data结构中；
// Note:    binder_transaction_data结构对象被封装进IPCThreadState::mOut
// Note:    IPCThreadState::mOut的数据地址被封装进**binder_write_read**中
// 上一次copy_from_user仅仅把binder_write_read结构对象拷贝进内核空间，但是IPCThreadState::mOut中的真正的数据部分并没有被拷贝
                if (copy_from_user(&tr, ptr, sizeof(tr)))
                    return -EFAULT;
                ptr += sizeof(tr);
                binder_transaction(proc, thread, &tr,
                        cmd == BC_REPLY, 0);
                break;
            }
            case BC_REGISTER_LOOPER:// ... 省略代码
            case BC_ENTER_LOOPER:// ... 省略代码
            case BC_EXIT_LOOPER:// ... 省略代码
            case BC_REQUEST_DEATH_NOTIFICATION:
            case BC_CLEAR_DEATH_NOTIFICATION:// ... 省略代码
            case BC_DEAD_BINDER_DONE:// ... 省略代码
            default:
                pr_err("%d:%d unknown command %d\n",
                    proc->pid, thread->pid, cmd);
                return -EINVAL;
            }
            *consumed = ptr - buffer;
        }
        return 0;
    }
```

&emsp;&emsp;&emsp;[binder_transaction][binder_transaction_lk]  

```c++
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

    static void binder_transaction(struct binder_proc *proc,
                    struct binder_thread *thread,
                    struct binder_transaction_data *tr, int reply, // Note: reply是上个函数中的cmd == BC_REPLY，所以reply为0
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
// Note: 获得**struct binder_node**结构对象 target_node
                target_node = context->binder_context_mgr_node;
                if (target_node)
// Note: 根据target_node获得**struct binder_proc**结构的对象target_proc，对应就是本次IPC通信的目标进程
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
        t = kzalloc(sizeof(*t), GFP_KERNEL);
// ... 省略代码
        tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
// ... 省略代码
// Note: 非oneway的通信方式，把当前thread保存到transaction的from字段
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
// Note: 从目标进程target_proc中分配内存空间
        t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
            tr->offsets_size, extra_buffers_size,
            !reply && (t->flags & TF_ONE_WAY), current->tgid);
// ... 省略代码
        t->buffer->debug_id = t->debug_id;
        t->buffer->transaction = t;
        t->buffer->target_node = target_node;
        t->buffer->clear_on_free = !!(t->flags & TF_CLEAR_BUF);
        trace_binder_transaction_alloc_buf(t->buffer);
// Note: binder_transaction_data.data.ptr.buffer就是名为data的Parcel对象中mData的值(reinterpret_cast<uintptr_t>(mData))，也就是数据的地址
// Note: binder_transaction_data.data_size则是mData对应的数据的大小(mDataSize > mDataPos ? mDataSize : mDataPos)，单位为size_t
        if (binder_alloc_copy_user_to_buffer(
                    &target_proc->alloc,
                    t->buffer, 0,
                    (const void __user *)
                        (uintptr_t)tr->data.ptr.buffer,
                    tr->data_size)) {
// ... 省略代码
            goto err_copy_data_failed;
        }
// Note: binder_transaction_data.data.ptr.offsets就是名为data的Parcel对象中所有flat_binder_object对象的起始地址(reinterpret_cast<uintptr_t>(mObjects))
// Note: binder_transaction_data.offsets_size则是这些flat_binder_object对象总的数据大小(data.ipcObjectsCount()*sizeof(binder_size_t))
        if (binder_alloc_copy_user_to_buffer(
                    &target_proc->alloc,
                    t->buffer,
                    ALIGN(tr->data_size, sizeof(void *)),
                    (const void __user *)
                        (uintptr_t)tr->data.ptr.offsets,
                    tr->offsets_size)) {
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
            if (!binder_proc_transaction(t, target_proc, target_thread)) {// ... 省略代码
            }
        } else {// ... 省略代码
            binder_enqueue_thread_work(thread, tcomplete);
            if (!binder_proc_transaction(t, target_proc, NULL))// ... 省略代码
        }
// ... 省略代码
        return;
// ... 省略代码
    }
```

&emsp;&emsp;&emsp;&emsp;[binder_get_node_refs_for_txn][binder_get_node_refs_for_txn_lk]  
&emsp;&emsp;&emsp;&emsp;[binder_alloc_new_buf][binder_alloc_new_buf_lk]  
&emsp;&emsp;&emsp;&emsp;&emsp;[binder_alloc_new_buf_locked][binder_alloc_new_buf_locked_lk]  
&emsp;&emsp;&emsp;&emsp;[binder_alloc_copy_user_to_buffer][binder_alloc_copy_user_to_buffer_lk]  

```c++
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

&emsp;&emsp;&emsp;&emsp;[binder_proc_transaction][binder_proc_transaction_lk]  

```c++
    static bool binder_proc_transaction(struct binder_transaction *t,
                        struct binder_proc *proc,
                        struct binder_thread *thread)
    {
        struct binder_node *node = t->buffer->target_node;
        bool oneway = !!(t->flags & TF_ONE_WAY);
        bool pending_async = false;
// ... 省略代码
// Note: 如果target_thread不为空，那么就把该任务（&t->work）加入到thread的todo列表中
        if (thread)
            binder_enqueue_thread_work_ilocked(thread, &t->work);
// Note: 反之，那么就把该任务（&t->work）加入到process的todo列表中
        else if (!pending_async)
            binder_enqueue_work_ilocked(&t->work, &proc->todo);
        else
            binder_enqueue_work_ilocked(&t->work, &node->async_todo);

        if (!pending_async)
            binder_wakeup_thread_ilocked(proc, thread, !oneway /* sync */);
// ... 省略代码
        return true;
    }
```

&emsp;&emsp;&emsp;&emsp;&emsp;[binder_wakeup_thread_ilocked][binder_wakeup_thread_ilocked_lk]  

```c++
    static void binder_wakeup_thread_ilocked(struct binder_proc *proc,
                        struct binder_thread *thread,
                        bool sync)
    {
        assert_spin_locked(&proc->inner_lock);

        if (thread) {
            if (sync)
// Note: 唤醒对应的线程处理
                wake_up_interruptible_sync(&thread->wait);
            else
                wake_up_interruptible(&thread->wait);
            return;
        }

        binder_wakeup_poll_threads_ilocked(proc, sync);
    }
```

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[wake_up_interruptible][wake_up_interruptible_lk]  

[binder_ioctl_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L4999
[binder_ioctl_write_read_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L4824
[binder_thread_write_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L3577
[binder_transaction_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L2814
[binder_get_node_refs_for_txn_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L2793
[binder_alloc_new_buf_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L567
[binder_alloc_new_buf_locked_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L378
[binder_alloc_copy_user_to_buffer_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L1199
[binder_proc_transaction_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L2727
[binder_wakeup_thread_ilocked_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L984
[wake_up_interruptible_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L994

### 5. 唤醒service manager线程之后再service manager的进程中执行

从第四篇我们可以知道，service manger进程中的Looper线程监听了binder驱动对应的设备文件节点。现在，上一小节已经写入了事件，接下来service manager进程被唤醒，开始调用被监听的binder驱动对应的设备文件节点对应的LooperCallback对象的BinderCallback::handleEvent函数：  

[BinderCallback::handleEvent][BinderCallbackHandleEventLink]  
[IPCThreadState::handlePolledCommands][IPCTSHandlePolledCommandsLink]  
[IPCThreadState::getAndExecuteCommand][IPCTSgetAndExecuteCommandLink]  
[IPCThreadState::executeCommand][IPCTSexecuteCommandLink]  
[BBinder::transact][BBinderTransactLink]  
[BnServiceManager::onTransact][BnServiceManagerOnTransactLink]  
[ServiceManager::addService][ServiceManagerAddServiceLink]  

```c++
Status ServiceManager::addService(const std::string& name, const sp<IBinder>& binder, bool allowIsolated, int32_t dumpPriority) {
    auto ctx = mAccess->getCallingContext();
// ... 省略代码

    // Overwrite the old service if it exists
    mNameToService[name] = Service {
        .binder = binder,
        .allowIsolated = allowIsolated,
        .dumpPriority = dumpPriority,
        .debugPid = ctx.debugPid,
    };

// ... 省略代码
    return Status::ok();
}
```

[BinderCallbackHandleEventLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/main.cpp;l=58
[IPCTSHandlePolledCommandsLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=670
[IPCTSgetAndExecuteCommandLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=516
[IPCTSexecuteCommandLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=1274
[BBinderTransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Binder.cpp;l=172
[BnServiceManagerOnTransactLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/native/libs/binder/libbinder_aidl_test_stub-cpp-source/gen/android/os/IServiceManager.cpp;l=501
[ServiceManagerAddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/ServiceManager.cpp;l=248

### 6. 下面回到驱动看看AMS线程中发生了什么

&emsp;&emsp;[binder_thread_read][binder_thread_read_lk]  

[binder_thread_read_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L4162

## 附

参考资料：[《Android Binder框架实现之Native层addService详解之请求的发送》](https://blog.csdn.net/tkwxty/article/details/103243685)
参考资料：[《Binder系列5—注册服务(addService)》](http://gityuan.com/2015/11/14/binder-add-service/)
参考资料：[《彻底理解Android Binder通信架构 3.5 binder_thread_read》](http://gityuan.com/2016/09/04/binder-start-service/#35-binder_thread_read)
参考资料：[《Binder內存拷貝的本質和變遷》](https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/741590/)
