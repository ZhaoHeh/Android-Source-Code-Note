# 一、Binder server端注册service的过程

TODO：驱动中的filp参数？  
TODO: IPCThreadState::mIn empty与否是由谁决定的？  

## AMS（ActivityManagerService）注册过程分析

[ActivityManagerService#setSystemProcess][AMSsetSystemProcessLink]  
&emsp;[ServiceManager#addService][SvcMgraddServiceLink]  
&emsp;&emsp;[ServiceManagerProxy#addService][SMPaddServiceLink]  
&emsp;&emsp;&emsp;[IServiceManager.Stub.Proxy#addService][ISMSPaddServiceLink]  

[AMSsetSystemProcessLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=2102
[SvcMgraddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=179
[SMPaddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=70
[ISMSPaddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=414

在第一篇已经分析过ServiceManager的实际组成，这里不在赘述。

### 1. 第一次封装，主要阐述Stub类型的system service作为一个参数的加工过程

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
```

```java
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

```c
/*
|----------------------------------------------------------------| <-------- android::Parcel::mData <~~ (android::Parcel) mNativePtr <~~ (android.os.Parcel) _data
|IServiceManager#DESCRIPTOR("android.os.IServiceManager")        | 
|----------------------------------------------------------------|
|Context.ACTIVITY_SERVICE("activity")                            |
|----------------------------------------------------------------|
|flat_binder_object(from ActivityManagerService)                 |
|----------------------------------------------------------------|
|int(allowIsolated true 1)                                       |
|----------------------------------------------------------------|
|int(DUMP_FLAG_PRIORITY_DEFAULT 8)                               |
|----------------------------------------------------------------|
*/
```

### 2. 第二次封装，BpBinder开始transact

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
// Note: cmd的实际值为 BC_TRANSACTION
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
// 代码路径：frameworks/native/libs/binder/IPCThreadState.cpp
```

对将要发送到Binder的数据进一步封装成**binder_transaction_data**结构类型，再将binder_transaction_data中的数据写入IPCThreadState::mOut中。  
说明：  
IPCThreadState::mOut：当前线程和Binder驱动通信时输入到Binder驱动中的数据；  
IPCThreadState::mIn：当前线程和Binder驱动通信时从Binder驱动中读出的数据。  

```c
/*
                                             |----------------------------------------| <-------- IPCThreadState::mOut::mData 
                                             | BC_TRANSACTION                         |
                                             |----------------------------------------| struct  binder_transaction_data
                                             | tr.target.ptr                          |
                                             |----------------------------------------|
                                             | tr.target.handle                       |
                                             |----------------------------------------|
                                             | tr.code                                |
                                             |----------------------------------------|
                                             | tr.flags                               |
            _data                            |----------------------------------------|
|--------------------------------| <-------- | tr.data.ptr.buffer                     |
|DESCRIPTOR                      |           |----------------------------------------|
|--------------------------------|           |......                                  |
|Context.ACTIVITY_SERVICE        |           |----------------------------------------|
|--------------------------------|
|flat_binder_object              |
|--------------------------------|
|int(allowIsolated true 1)       |
|--------------------------------|
|int(DUMP_FLAG_PRIORITY_DEFAULT 8)|
|--------------------------------|
*/
```

### 3 第三次封装，进程和驱动通信

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
[talkWithDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=971
[IOCTLSystemCallDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=1018

在IPCThreadState::talkWithDriver中做了第三次也是最后一次封装，将IPCThreadState::mOut和IPCThreadState::mIn封装到**binder_write_read**结构中；  
将binder_write_read结构在用户空间的虚拟地址通过ioctl传递到驱动中，数据最终以**binder_write_read**的格式由客户端与驱动交互。  

```c
/*

                                                                                                        struct binder_write_read
                                                                                                    |--------------------------------| <-------- &bwr/void __user *ubuf/(void __user *)arg
                                                                                                    |  binder_size_t write_size;     |
                                                                                                    |--------------------------------|
                                                                                                    |  binder_size_t write_consumed; |
                                                                                                    |--------------------------------|
                                                                                                ----|  binder_uintptr_t write_buffer;|
                                                                                                |   |--------------------------------|
                                                                                                |   |  binder_size_t read_size;      |
                                                                                                |   |--------------------------------|
                                                                                                |   |  binder_size_t read_consumed;  |
                                                                                                |   |--------------------------------|
                                                                                                | --|  binder_uintptr_t read_buffer; |
                                                    IPCThreadState::mOut::mData                 | | |--------------------------------|
                                             |----------------------------------------| <-------  |
                                             |......                                  |           |
                                             |----------------------------------------|           |
                                             |binder_transaction_data.target.ptr      |           |
                                             |----------------------------------------|           |
                                             |binder_transaction_data.target.handle   |           |
                                             |----------------------------------------|           |
                                             |binder_transaction_data.code            |           |
                                             |----------------------------------------|           |
                                             |binder_transaction_data.flags           |           |
            _data                            |----------------------------------------|           |
|--------------------------------| <-------- |binder_transaction_data.data.ptr.buffer |           |
|DESCRIPTOR                      |           |----------------------------------------|           |
|--------------------------------|           |......                                  |           |
|Context.ACTIVITY_SERVICE        |           |----------------------------------------|           |
|--------------------------------|                                                                |
|flat_binder_object              |                                                 null <---------|
|--------------------------------|
|int(allowIsolated true 1)       |
|--------------------------------|
|int(DUMP_FLAG_PRIORITY_DEFAULT 8)|
|--------------------------------|
*/
```

### 4. 下面进入驱动执行

[binder_ioctl][binder_ioctl_lk]  

```c++
// Note: 参数说明
// Note: filp   文件指针，Linux系统调用通过struct file实现驱动函数的映射？？？？？？
// Note: cmd    BINDER_WRITE_READ
// Note: arg    用户空间中binder_write_read结构对象的地址
    static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
    {
        int ret;
// Note: 获得该进程在Binder驱动中对应的结构体
        struct binder_proc *proc = filp->private_data;
        struct binder_thread *thread;
        unsigned int size = _IOC_SIZE(cmd);
// Note: ubuf是一个指针变量，值等于unsigned long类型的arg，arg就是用户空间bwr结构的指针，所以ubuf的值就等于bwr结构在用户空间的虚拟地址。
        void __user *ubuf = (void __user *)arg;
        // ... 省略代码
        // Note: wait_event_interruptible 当 binder_stop_on_user_error >= 2时，会令当前线程进入休眠状态(wait_event_interruptible的具体原理参考第六篇)
        // Note: binder_stop_on_user_error 的值的修改应该和binder module在内核的加载有关，因此此处的 wait_event_interruptible 应该是一种错误处理，不必太关注
        ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
        // ... 省略代码
// Note: 查找或创建当前线程对应的**struct binder_thread**结构
        thread = binder_get_thread(proc);
        // ... 省略代码
        switch (cmd) {
// Note: 根据参数cmd的值找到对应的处理函数
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
        // ... 省略代码
        return ret;
    }
// 总结：binder_ioctl函数仅仅对传递过来的参数进行简单的处理，获取到一些关键信息：
//          client端在binder驱动中对应的“进程结构体”、“线程结构体”、命令以及struct binder_write_read数据包在用户空间的地址
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
// Note: 在内核空间binder_ioctl_write_read函数的栈中开辟一个临时内存，大小为sizeof(struct binder_write_read)，用&bwr指示
        struct binder_write_read bwr;
        // ... 省略代码
        if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
// Note: 从用户空间ubuf指示的位置，拷贝sizeof(bwr)大小的数据到内核空间&bwr指示的位置，也就是传说中Binder一次拷贝的发生处，同时我们需要注意，这次拷贝只是把如下这段内存拷了进来：
// Note: write_buffer和read_buffer作为指针变量指示的用户空间的内存并没有被拷贝
/*
             struct binder_write_read
        |--------------------------------| <-------- &bwr（内核空间）
        |  binder_size_t write_size;     |
        |--------------------------------|
        |  binder_size_t write_consumed; |
        |--------------------------------|
        |  binder_uintptr_t write_buffer;|
        |--------------------------------|
        |  binder_size_t read_size;      |
        |--------------------------------|
        |  binder_size_t read_consumed;  |
        |--------------------------------|
        |  binder_uintptr_t read_buffer; |
        |--------------------------------|
*/
            // ... 省略错误处理代码
        }
        // ... 省略代码
        if (bwr.write_size > 0) {
// Note: bwr.write_buffer 对应的就是 IPCThreadState::mOut 的数据地址，也就是 IPCThreadState::mOut::mData 指针变量所指的地址
// Note: bwr.write_size 就是 IPCThreadState::mOut 中数据的大小
// Note: bwr.write_consumed 就是在用户空间talkWithDriver中设置为0的，标志已经write的数据量的大小的变量
            ret = binder_thread_write(proc, thread,
                        bwr.write_buffer,
                        bwr.write_size,
                        &bwr.write_consumed);
            // ... 省略错误处理代码
        }
        if (bwr.read_size > 0) {
// Note: bwr.read_buffer 对应的就是 IPCThreadState::mIn 的数据地址，也就是 IPCThreadState::mIn::mData 指针变量所指的地址
// Note: bwr.read_size 则是 IPCThreadState::mIn 中数据的大小
// Note: bwr.read_consumed 初始值会被设为0，然后在驱动中被修改
// Note: filp->f_flags包括O_RDONLY, O_NONBLOCK, and O_SYNC三个flag，通常驱动里面需要检查O_NONBLOCK，判断是否为不可阻塞的操作，可以不用关注
            ret = binder_thread_read(proc, thread, bwr.read_buffer,
                        bwr.read_size,
                        &bwr.read_consumed,
                        filp->f_flags & O_NONBLOCK);
            // ... 省略错误处理代码
        }
        // ... 省略代码
        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
// Note: 最后将修改完的数据拷贝回用户空间
/*
                             struct binder_write_read
ubuf(user space) --------> |--------------------------------|
                           |  binder_size_t write_size;     |
                           |--------------------------------|
                           |  binder_size_t write_consumed; |
                           |--------------------------------|
                           |  binder_uintptr_t write_buffer;|
                           |--------------------------------|
                           |  binder_size_t read_size;      |
                           |--------------------------------|
                           |  binder_size_t read_consumed;  |
                           |--------------------------------|
                           |  binder_uintptr_t read_buffer; |
                           |--------------------------------|
*/
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
// Note: 下面开始拷贝用户空间指针ptr指示的内容，此时是之前装入的 BC_TRANSACTION，并将值赋给binder_thread_write函数内的局部变量uint32_t cmd；
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
// Note: 在内核空间 binder_thread_write 函数的栈中开辟一个临时内存，大小为sizeof(struct binder_transaction_data)，并用&tr指示
                struct binder_transaction_data tr;
                if (copy_from_user(&tr, ptr, sizeof(tr)))
// Note: 首先这里要复习一下数据在用户态的封装: 
// Note:    Parcel类型的data封装了addService的入参，data作为指针变量又被封装进 binder_transaction_data 结构中；
// Note:    binder_transaction_data 结构对象被封装进 IPCThreadState::mOut
// Note:    IPCThreadState::mOut的数据地址被封装进**binder_write_read**中
// Note: 上一次copy_from_user仅仅把binder_write_read结构对象拷贝进内核空间，但是IPCThreadState::mOut中的真正的数据部分并没有被拷贝，本次完成了struct binder_transaction_data数据的拷贝：
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
                    return -EFAULT;
                ptr += sizeof(tr);
// Note: 根据cmd确认这是一次transaction后，拷贝完成上述结构，然后将数据地址&tr传入binder_transaction，进入transact：
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
            // ... 省略代码
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
// Note: binder_alloc_new_buf -- Very Very Very Important!!!
// Note:     target_proc->alloc是一个*struct binder_alloc*类型的成员变量，管理着目标进程（本次transaction对应的则是service manager）曾经通过mmap映射的那段内核和用户空间都可以看到的内存
// Note:     这段内存被分成了一段段，每一段都用一个*struct binder_buffer*对象来管理，*struct binder_alloc*则管理这些*struct binder_buffer*
// Note:     所谓binder_alloc_new_buf实际上并没有真正allocate，而是在target_proc->alloc管理的众多*struct binder_buffer*中找到最合适的一个，然后返回其指针
        t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
            tr->offsets_size, extra_buffers_size,
            !reply && (t->flags & TF_ONE_WAY), current->tgid);
        // ... 省略代码
        t->buffer->debug_id = t->debug_id;
        t->buffer->transaction = t;
        t->buffer->target_node = target_node;
        t->buffer->clear_on_free = !!(t->flags & TF_CLEAR_BUF);
// Note: 以上操作就是简单地对这个*struct binder_buffer*初始化
        // ... 省略代码
        if (binder_alloc_copy_user_to_buffer(
                    &target_proc->alloc,
                    t->buffer, 0,
                    (const void __user *)
                        (uintptr_t)tr->data.ptr.buffer,
                    tr->data_size)) {
// Note: binder_transaction_data.data.ptr.buffer 就是名为data的Parcel对象中mData的值(reinterpret_cast<uintptr_t>(mData))，也就是数据的地址
// Note: binder_transaction_data.data_size 则是mData对应的数据的大小(mDataSize > mDataPos ? mDataSize : mDataPos)，单位为size_t
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
// Note: binder_transaction_data.offsets_size则是这些flat_binder_object对象总的数据大小(data.ipcObjectsCount()*sizeof(binder_size_t))
            // ... 省略代码
            goto err_copy_data_failed;
        }
// Note: 上面两个 binder_alloc_copy_user_to_buffer 函数分别拷贝了_data中普通数据和flat_binder_object类型对象，因此可以放到一起看
// Note: 最终完成的就是将_data的数据拷贝到映射的内存中，之后服务端使用是就不用再拷贝，这就是为什么binder是用一次拷贝完成了跨进程的通信。
/*
(struct binder_transaction *)t->(struct binder_buffer *)buffer->(void __user *)user_data --|
                                                                                           |
 |-----------------------------------------------------------------------------------------|
 |
 |--------> |----------------------------------------------------------------|
            |IServiceManager#DESCRIPTOR("android.os.IServiceManager")        | 
            |----------------------------------------------------------------|
            |Context.ACTIVITY_SERVICE("activity")                            |
            |----------------------------------------------------------------|
            |flat_binder_object(from ActivityManagerService)                 |
            |----------------------------------------------------------------|
            |int(allowIsolated true 1)                                       |
            |----------------------------------------------------------------|
            |int(DUMP_FLAG_PRIORITY_DEFAULT 8)                               |
            |----------------------------------------------------------------|
    
*/
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
[binder_proc_transaction_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L2727
[binder_wakeup_thread_ilocked_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L984
[wake_up_interruptible_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L994

### 进入service manager进程前的一些问题

通过查看*service manager*进程进入loop的过程可以发现，*service manager*进程通过调用[Looper::pollAll][main_call_pollAll_link]开始监听[binder_fd][addFd_binder_fd_link]，通过查阅代码，这个流程最终会使用[epoll_wait][use_epoll_wait_link]监听。  
等到有事件到来，则会调用已注册的[callback][call_registered_cb_link]执行对应的任务。根据frameworks/native/cmds/servicemanager/main.cpp中的注册流程，可知此时调用的callback具体为[BinderCallback::handleEvent][the_target_cb_link]。进一步追踪代码流程，可知最终会通过[IPCThreadState::getAndExecuteCommand][getAndExecuteCommandLink]调用[IPCThreadState::talkWithDriver][talkWithDriverLink]，根据前面的代码分析可知，最终会进入驱动的[binder_thread_read][binder_thread_read_lk]执行。  
现在的问题则是，上一节描述的客户端进程（也就是ActivityServiceManager）在内核态通过函数[binder_wakeup_thread_ilocked][binder_wakeup_thread_ilocked_lk]唤醒了*service manager*进程的binder线程，触发了*service manager*进程的哪一个节点的执行呢？是结束了在[epoll_wait][use_epoll_wait_link]中的等待呢还是结束了在[binder_thread_read][binder_thread_read_lk]的等待然后向下执行的？  

[main_call_pollAll_link]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/main.cpp;l=137
[addFd_binder_fd_link]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/main.cpp;l=48
[use_epoll_wait_link]:https://cs.android.com/android/platform/superproject/+/master:system/core/libutils/Looper.cpp;l=239
[call_registered_cb_link]:https://cs.android.com/android/platform/superproject/+/master:system/core/libutils/Looper.cpp;l=355
[the_target_cb_link]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/main.cpp;l=58
[getAndExecuteCommandLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=501
[talkWithDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=971
[binder_thread_read_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L3775
[binder_wakeup_thread_ilocked_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L994

### 6. 在service manager进程的内核态执行

不管上一小节的问题答案最终是什么，我们知道service manager进程总是该从*binder_thread_read*函数开始：  

[binder_ioctl_write_read][binder_ioctl_write_read_lk]  

```c++
    static int binder_ioctl_write_read(struct file *filp,
                    unsigned int cmd, unsigned long arg,
                    struct binder_thread *thread)
    {
        int ret = 0;
        struct binder_proc *proc = filp->private_data;
        unsigned int size = _IOC_SIZE(cmd);
        void __user *ubuf = (void __user *)arg;
        struct binder_write_read bwr; // ... 省略代码

// Note: 仍旧是上面提到过得第一次拷贝，如下结构拷贝如内核空间
/*
                   struct binder_write_read
&bwr --------> |--------------------------------|
               |  binder_size_t write_size;     |
               |--------------------------------|
               |  binder_size_t write_consumed; |
               |--------------------------------|
               |  binder_uintptr_t write_buffer;|
               |--------------------------------|
               |  binder_size_t read_size;      |
               |--------------------------------|
               |  binder_size_t read_consumed;  |
               |--------------------------------|
               |  binder_uintptr_t read_buffer; |
               |--------------------------------|
*/
        if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
            ret = -EFAULT;
            goto out;
        }
        // ... 省略错误处理代码

        if (bwr.write_size > 0) {
            // ... 省略 binder_thread_write 代码
        }
        if (bwr.read_size > 0) {
            ret = binder_thread_read(proc, thread, bwr.read_buffer,
                        bwr.read_size,
                        &bwr.read_consumed,
                        filp->f_flags & O_NONBLOCK);
            // ... 省略代码
        }
        // ... 省略错误处理代码
        if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
            ret = -EFAULT;
            goto out;
        }
    out:
        return ret;
    }
```

&emsp;&emsp;[binder_thread_read][binder_thread_read_lk]  

```c++
    static int binder_thread_read(struct binder_proc *proc,
                    struct binder_thread *thread,
                    binder_uintptr_t binder_buffer, size_t size,
                    binder_size_t *consumed, int non_block)
    {
// Note: buffer 是一个指针变量，指向的是该进程用户空间 IPCThreadState::mIn::mData 指向的地址
// Note: ptr 是一个指针变量，指向的是当前将要写入的位置，由数据首地址 buffer + 偏移量 *consumed 得到
// Note: end 依旧是一个指针变量，指向的是数据的尾地址
        void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
        void __user *ptr = buffer + *consumed;
        void __user *end = buffer + size;

        int ret = 0;
        int wait_for_proc_work;

        if (*consumed == 0) {
// Note: ？？？？？？ 这段代码的意义是什么? 每次binder_thread_read都会先执行这个操作吗？这个BR_NOOP命令和BR_TRANSACTION命令应该是互斥的，那么BR_TRANSACTION如何替换它呢？？？
            if (put_user(BR_NOOP, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
        }

    retry:
        binder_inner_proc_lock(proc);
// Note: wait_for_proc_work 实际上充当的是bool变量，表示是否要等待work的到来；
// Note: 如果 thread->transaction_stack 不为空，thread->todo 为空，且 thread->looper 的 BINDER_LOOPER_STATE_REGISTERED 位 或 BINDER_LOOPER_STATE_ENTERED 位 不为空
// Note: 则 wait_for_proc_work 为true
        wait_for_proc_work = binder_available_for_proc_work_ilocked(thread);
        binder_inner_proc_unlock(proc);

        thread->looper |= BINDER_LOOPER_STATE_WAITING;

        // ... 省略代码
        if (wait_for_proc_work) {
            // ... 省略代码
            binder_set_nice(proc->default_priority);
        }

        if (non_block) {
            if (!binder_has_work(thread, wait_for_proc_work))
                ret = -EAGAIN;
        } else {
            ret = binder_wait_for_work(thread, wait_for_proc_work);
        }

        thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

        if (ret)
            return ret;

        while (1) {
            uint32_t cmd;
// Note: 
/*
    struct binder_transaction_data_secctx {
        struct binder_transaction_data transaction_data;
        binder_uintptr_t secctx;
    };
*/
            struct binder_transaction_data_secctx tr;
            struct binder_transaction_data *trd = &tr.transaction_data;
            struct binder_work *w = NULL;
            struct list_head *list = NULL;
            struct binder_transaction *t = NULL;
            struct binder_thread *t_from;
            size_t trsize = sizeof(*trd);

            binder_inner_proc_lock(proc);
// Note: 下面这段逻辑就是获取todo work的列表以及错误处理
            if (!binder_worklist_empty_ilocked(&thread->todo))
                list = &thread->todo;
            else if (!binder_worklist_empty_ilocked(&proc->todo) &&
                wait_for_proc_work)
                // ... 省略代码
            else {// ... 省略代码
            }

            // ... 省略代码
// Note: 从列表中获取第一个work
            w = binder_dequeue_work_head_ilocked(list);
            if (binder_worklist_empty_ilocked(&thread->todo))
                thread->process_todo = false;

            switch (w->type) {
            case BINDER_WORK_TRANSACTION: {
// Note: 通过binder_work类型的指针变量w获取binder_work的container -- binder_transaction结构，并用指针t指示
                binder_inner_proc_unlock(proc);
                t = container_of(w, struct binder_transaction, work);
            } break;
            case BINDER_WORK_RETURN_ERROR: // ... 省略代码
            case BINDER_WORK_TRANSACTION_COMPLETE: // ... 省略代码
            case BINDER_WORK_NODE: // ... 省略代码
            case BINDER_WORK_DEAD_BINDER:
            case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
            case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: // ... 省略代码
            default: // ... 省略代码
            }

            if (!t)
                continue;

            // ... 省略错误处理的代码
// Note: !!!!!!
// Note: 获取目标node
            if (t->buffer->target_node) {
                struct binder_node *target_node = t->buffer->target_node;

                trd->target.ptr = target_node->ptr;
                trd->cookie =  target_node->cookie;
                t->saved_priority = task_nice(current);
                if (t->priority < target_node->min_priority &&
                    !(t->flags & TF_ONE_WAY))
                    binder_set_nice(t->priority);
                else if (!(t->flags & TF_ONE_WAY) ||
                    t->saved_priority > target_node->min_priority)
                    binder_set_nice(target_node->min_priority);
// Note: 将命令设置为BR_TRANSACTION，后面还会看到
                cmd = BR_TRANSACTION;
            } else {
                trd->target.ptr = 0;
                trd->cookie = 0;
                cmd = BR_REPLY;
            }
            trd->code = t->code;
            trd->flags = t->flags;
            trd->sender_euid = from_kuid(current_user_ns(), t->sender_euid);

            t_from = binder_get_txn_from(t);
            if (t_from) {
                struct task_struct *sender = t_from->proc->tsk;

                trd->sender_pid =
                    task_tgid_nr_ns(sender,
                            task_active_pid_ns(current));
            } else {
                trd->sender_pid = 0;
            }

            ret = binder_apply_fd_fixups(proc, t);
            if (ret) {
                struct binder_buffer *buffer = t->buffer;
                bool oneway = !!(t->flags & TF_ONE_WAY);
                int tid = t->debug_id;

                if (t_from)
                    binder_thread_dec_tmpref(t_from);
                buffer->transaction = NULL;
                binder_cleanup_transaction(t, "fd fixups failed",
                            BR_FAILED_REPLY);
                binder_free_buf(proc, buffer);
                binder_debug(BINDER_DEBUG_FAILED_TRANSACTION,
                        "%d:%d %stransaction %d fd fixups failed %d/%d, line %d\n",
                        proc->pid, thread->pid,
                        oneway ? "async " :
                        (cmd == BR_REPLY ? "reply " : ""),
                        tid, BR_FAILED_REPLY, ret, __LINE__);
                if (cmd == BR_REPLY) {
                    cmd = BR_FAILED_REPLY;
                    if (put_user(cmd, (uint32_t __user *)ptr))
                        return -EFAULT;
                    ptr += sizeof(uint32_t);
                    binder_stat_br(proc, thread, cmd);
                    break;
                }
                continue;
            }
            trd->data_size = t->buffer->data_size;
            trd->offsets_size = t->buffer->offsets_size;
// Note: 这里的赋值 Very Very Very Important: 
            trd->data.ptr.buffer = (uintptr_t)t->buffer->user_data;
// Note:
/*
(struct binder_transaction_data_secctx)tr.(struct binder_transaction_data)transaction_data ----|
                                                                                               |
|----------------------------------------------------------------------------------------------|
|
|--------> |----------------------|
           | trd.target           |
           |----------------------|
           | trd.cookie           |
           |----------------------|
           | trd.code             |
           |----------------------|
           | trd.flags            |
           |----------------------|
           | trd.sender_pid       |
           |----------------------|
           | trd.sender_euid      |
           |----------------------|
           | trd.data_size        |
           |----------------------|
           | trd.offsets_size     |
           |----------------------|               server端mmap的空间
           | trd.data.ptr.buffer  | --------> |--------------------------------| <-------- (struct binder_transaction *)t->(struct binder_buffer *)buffer->(void __user *)user_data
           |----------------------|           |DESCRIPTOR                      |
           | trd.data.ptr.offsets |           |--------------------------------|
           |----------------------|           |Context.ACTIVITY_SERVICE        |
                                              |--------------------------------|
                                              |flat_binder_object              |
                                              |--------------------------------|
                                              |int(allowIsolated true 1)       |
                                              |--------------------------------|
                                              |int(DUMP_FLAG_PRIORITY_DEFAULT 8)|
                                              |--------------------------------|
*/
            trd->data.ptr.offsets = trd->data.ptr.buffer +
                        ALIGN(t->buffer->data_size,
                            sizeof(void *));

            tr.secctx = t->security_ctx;
            if (t->security_ctx) {
                cmd = BR_TRANSACTION_SEC_CTX;
                trsize = sizeof(tr);
            }
            if (put_user(cmd, (uint32_t __user *)ptr)) {
// Note: 将cmd写回到用户空间
/*
                   struct binder_write_read
&bwr --------> |--------------------------------|
               |  binder_size_t write_size;     |
               |--------------------------------|
               |  binder_size_t write_consumed; |
               |--------------------------------|
               |  binder_uintptr_t write_buffer;|
               |--------------------------------|
               |  binder_size_t read_size;      |
               |--------------------------------|
               |  binder_size_t read_consumed;  |
               |--------------------------------|           用户空间IPCThreadState::mIn::mData
               |  binder_uintptr_t read_buffer; |--------> |--------------------------------|
               |--------------------------------|          | BR_TRANSACTION                 |
                                                           |--------------------------------|
*/
                // ... 省略错误处理代码
            }
            ptr += sizeof(uint32_t);
            if (copy_to_user(ptr, &tr, trsize)) {
// Note: unsigned long copy_to_user(void __user *to, const void *from, unsigned long n)：
// Note:    to 用户空间的指针
// Note:    from 内核空间的指针
// Note:    n 是内核空间向用户空间拷贝的字节数
// Note: 本函数将binder_thread_read栈中的临时变量struct binder_transaction_data_secctx类型的tr中的内容拷贝到用户空间
/*
                   struct binder_write_read
&bwr --------> |--------------------------------|
               |  binder_size_t write_size;     |
               |--------------------------------|
               |  binder_size_t write_consumed; |
               |--------------------------------|
               |  binder_uintptr_t write_buffer;|
               |--------------------------------|
               |  binder_size_t read_size;      |
               |--------------------------------|
               |  binder_size_t read_consumed;  |
               |--------------------------------|                  server端用户空间
               |  binder_uintptr_t read_buffer; |--------> |--------------------------------| <-------- IPCThreadState::mIn::mData
               |--------------------------------|          | BR_TRANSACTION                 |
(struct binder_transaction_data_secctx)tr.transaction_data |--------------------------------|
                                                           | trd.target                     |
                                                           |--------------------------------|
                                                           | trd.cookie                     |
                                                           |--------------------------------|
                                                           | trd.code                       |
                                                           |--------------------------------|
                                                           | trd.flags                      |
                                                           |--------------------------------|
                                                           | trd.sender_pid                 |
                                                           |--------------------------------|
                                                           | trd.sender_euid                |
                                                           |--------------------------------|
                                                           | trd.data_size                  |
                                                           |--------------------------------|
                                                           | trd.offsets_size               |
                                                           |--------------------------------|             server端mmap的空间
                                                           | trd.data.ptr.buffer            | --------> |--------------------------------|
                                                           |--------------------------------|           |DESCRIPTOR                      |
                                                           | trd.data.ptr.offsets           |           |--------------------------------|
                                                           |--------------------------------|           |Context.ACTIVITY_SERVICE        |
          (struct binder_transaction_data_secctx)tr.secctx | binder_uintptr_t secctx        |           |--------------------------------|
                                                           |--------------------------------|           |flat_binder_object              |
                                                                                                        |--------------------------------|
                                                                                                        |int(allowIsolated true 1)       |
                                                                                                        |--------------------------------|
                                                                                                        |int(DUMP_FLAG_PRIORITY_DEFAULT 8)|
                                                                                                        |--------------------------------|
*/
                // ... 省略错误处理代码
            }
            ptr += trsize;

            trace_binder_transaction_received(t);
            binder_stat_br(proc, thread, cmd);
            // ... 省略代码

            if (t_from)
                binder_thread_dec_tmpref(t_from);
            t->buffer->allow_user_free = 1;
            if (cmd != BR_REPLY && !(t->flags & TF_ONE_WAY)) {
                binder_inner_proc_lock(thread->proc);
                t->to_parent = thread->transaction_stack;
                t->to_thread = thread;
                thread->transaction_stack = t;
                binder_inner_proc_unlock(thread->proc);
            } else {
                binder_free_transaction(t);
            }
            break;
        }

    done:

        *consumed = ptr - buffer;
        binder_inner_proc_lock(proc);
        if (proc->requested_threads == 0 &&
            list_empty(&thread->proc->waiting_threads) &&
            proc->requested_threads_started < proc->max_threads &&
            (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
            BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
            /*spawn a new thread if we leave this out */) {
            proc->requested_threads++;
            binder_inner_proc_unlock(proc);
            binder_debug(BINDER_DEBUG_THREADS,
                    "%d:%d BR_SPAWN_LOOPER\n",
                    proc->pid, thread->pid);
            if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
                return -EFAULT;
            binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
        } else
            binder_inner_proc_unlock(proc);
        return 0;
    }
```

&emsp;&emsp;&emsp;[binder_available_for_proc_work_ilocked][binder_available_for_proc_work_lk]  

```c++
    static bool binder_available_for_proc_work_ilocked(struct binder_thread *thread)
    {
        return !thread->transaction_stack &&
            binder_worklist_empty_ilocked(&thread->todo) &&
            (thread->looper & (BINDER_LOOPER_STATE_ENTERED |
                    BINDER_LOOPER_STATE_REGISTERED));
    }
// 代码路径：/drivers/android/binder.c
    static bool binder_worklist_empty_ilocked(struct list_head *list)
    {
        return list_empty(list);
    }
// 代码路径：/drivers/android/binder.c
    static inline bool
    list_empty(struct list_head *head)
    {
        return head->next == head;
    }
// 代码路径：/drivers/gpu/drm/nouveau/include/nvif/list.h
```

&emsp;&emsp;&emsp;[binder_wait_for_work][binder_wait_for_work_lk]  

```c++
    static int binder_wait_for_work(struct binder_thread *thread,
                    bool do_proc_work)
    {
        DEFINE_WAIT(wait);
        struct binder_proc *proc = thread->proc;
        int ret = 0;

        freezer_do_not_count();
        binder_inner_proc_lock(proc);
        for (;;) {
            prepare_to_wait(&thread->wait, &wait, TASK_INTERRUPTIBLE);
            if (binder_has_work_ilocked(thread, do_proc_work))
                break;
            if (do_proc_work)
                list_add(&thread->waiting_thread_node,
                    &proc->waiting_threads);
            binder_inner_proc_unlock(proc);
            schedule();
            binder_inner_proc_lock(proc);
            list_del_init(&thread->waiting_thread_node);
            if (signal_pending(current)) {
                ret = -ERESTARTSYS;
                break;
            }
        }
        finish_wait(&thread->wait, &wait);
        binder_inner_proc_unlock(proc);
        freezer_count();

        return ret;
    }
// 代码路径：/drivers/android/binder.c
```

[binder_ioctl_write_read_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L3775
[binder_thread_read_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L4162
[binder_available_for_proc_work_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L508
[binder_wait_for_work_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L3678

### 7. 唤醒service manager线程之后在service manager的进程中执行

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

### 8. 下面回到驱动看看AMS线程中发生了什么

## 附

参考资料：[《Android Binder框架实现之Native层addService详解之请求的发送》](https://blog.csdn.net/tkwxty/article/details/103243685)  
参考资料：[《Binder系列5—注册服务(addService)》](http://gityuan.com/2015/11/14/binder-add-service/)  
参考资料：[《彻底理解Android Binder通信架构 3.5 binder_thread_read》](http://gityuan.com/2016/09/04/binder-start-service/#35-binder_thread_read)  
参考资料：[《Binder內存拷貝的本質和變遷》](https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/741590/)  
