# service manager进程

## 一、

**service manager**是一个native code实现的守护进程，我们直接看rc文件和main函数：  

```rc
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart apexd
    onrestart restart audioserver
    onrestart restart gatekeeperd
    onrestart class_restart main
    onrestart class_restart hal
    onrestart class_restart early_hal
    writepid /dev/cpuset/system-background/tasks
    shutdown critical
// 代码路径：frameworks/native/cmds/servicemanager/servicemanager.rc
```

```c++
int main(int argc, char** argv) {
    if (argc > 2) {
        LOG(FATAL) << "usage: " << argv[0] << " [binder driver]";
    }

    const char* driver = argc == 2 ? argv[1] : "/dev/binder";

    sp<ProcessState> ps = ProcessState::initWithDriver(driver);
    ps->setThreadPoolMaxThreadCount(0);
    ps->setCallRestriction(ProcessState::CallRestriction::FATAL_IF_NOT_ONEWAY);

    sp<ServiceManager> manager = new ServiceManager(std::make_unique<Access>());
    if (!manager->addService("manager", manager, false /*allowIsolated*/, IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) {
        LOG(ERROR) << "Could not self register servicemanager";
    }

    IPCThreadState::self()->setTheContextObject(manager);
    ps->becomeContextManager();

    sp<Looper> looper = Looper::prepare(false /*allowNonCallbacks*/);

    BinderCallback::setupTo(looper);
    ClientCallbackCallback::setupTo(looper, manager);

    while(true) {
        looper->pollAll(-1);
    }

    // should not be reached
    return EXIT_FAILURE;
}
// 代码路径：frameworks/native/cmds/servicemanager/main.cpp
```

service manager也是一个Binder IPC的服务端，因此**android::ServiceManager**类肯定是一个BBinder，我们看一下他的继承关系：  
android::ServiceManager -> android::os::BnServiceManager\<IServiceManager\>(Generated) -> android::BBinder，果然是这样的。  

## 二、

&emsp;[ProcessState::initWithDriver][PSinitWithDriverLink]  
&emsp;&emsp;[ProcessState::init][PSinitLink]  
&emsp;&emsp;&emsp;[ProcessState::ProcessState][PSConstructorLink]  
&emsp;&emsp;&emsp;&emsp;[ProcessState::open_driver][PSopen_driverLink]  
&emsp;&emsp;&emsp;&emsp;[mmap][mmapCalledLink]  

[PSinitWithDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=80
[PSinitLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=90
[PSConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=395
[PSopen_driverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=367
[mmapCalledLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=395

```c++
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
// Note: mmap系统调用的函数声明为：
// Note:     void * mmap(void *start, size_t length, int prot , int flags, int fd, off_t offset)
// Note: 解释说明：
// Note:    start 表示要映射到的用户空间的内存区域的起始地址，nullptr表示由内核来指定
// Note:    length 表示要映射的内存区域的大小，此处BINDER_VM_SIZE为宏定义，实际预处理中替换为 ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)，sysconf(_SC_PAGE_SIZE)是Linux中的系统调用，返回的是
//                 Size of a page in bytes. 所以BINDER_VM_SIZE应该是 1MB - 2 * 页的大小
// Note:    prot 期望的内存保护标志，PROT_READ表示页内容可以被读取
// Note:    flags 指定映射对象（也就是目标文件，本例中为/dev/binder）的类型，MAP_PRIVATE表示建立一个写入时拷贝的私有映射，内存区域的写入不会影响到原文件；MAP_NORESERVE ：不要为这个映射保留交换空间。
// Note:    fd 文件描述符
// Note:    offset 表示被映射对象（也就是目标文件，本例中为/dev/binder）从哪里开始对应，单位是byte，该值应该是页大小的整数倍
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }
```

在调用到*file_operations.mmap*指定的操作之前，系统调用mmap做了如下工作：  
（1）在当前进程的虚拟地址空间中（也就是栈顶之下，堆顶之上的空间）寻找一段空闲的满足要求的连续虚拟地址  
（2）为此虚拟区域创建一个*vm_area_struct*结构，对这个结构的各个字段进行初始化，以后这段连续的虚拟地址空间就用这个结构对象表示了
（3）将新建的虚拟区结构对象插入进程的Virtual Memory Area链表中

以上工作完成后，系统调用找到对应的文件结构体（*struct file*），在文件结构体中找到*file_operations*模块，进而通过*file_operations.mmap*字段找到对应的操作，由于Binder驱动并非真正的磁盘文件而是一个驱动，而且还是一个基于内存的驱动，连实际的外设都不涉及，所以下面的映射工作针对内存的操作：  

下面进入驱动看看发生了什么：  

[binder_mmap][binder_mmap_lk]  

[binder_mmap_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L4780

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

    static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
    {
        struct binder_proc *proc = filp->private_data;

        // ... 省略错误处理的各种代码

        vma->vm_flags |= VM_DONTCOPY | VM_MIXEDMAP;
        vma->vm_flags &= ~VM_MAYWRITE;

        vma->vm_ops = &binder_vm_ops;
        vma->vm_private_data = proc;

        return binder_alloc_mmap_handler(&proc->alloc, vma);
    }

    static const struct vm_operations_struct binder_vm_ops = {
        .open = binder_vma_open,
        .close = binder_vma_close,
        .fault = binder_vm_fault,
    };
// 代码路径：/drivers/android/binder.c
```

&emsp;[binder_alloc_mmap_handler][binder_alloc_mmap_handler_lk]  

[binder_alloc_mmap_handler_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder_alloc.c#L741

```c++
    int binder_alloc_mmap_handler(struct binder_alloc *alloc,
                    struct vm_area_struct *vma)
    {
        int ret;
        const char *failure_string;
        struct binder_buffer *buffer;

        mutex_lock(&binder_alloc_mmap_lock);
// Note: 错误处理：如果当前进程的binder_alloc结构对象的binder_alloc.buffer_size > 0，则已经发生过映射
        if (alloc->buffer_size) {
            ret = -EBUSY;
            failure_string = "already mapped";
            goto err_already_mapped;
        }
// Note: 真正的binder_alloc.buffer_size还是很节制的，在 vma->vm_end - vma->vm_start 和 SZ_4M 之间去了一个最小值
        alloc->buffer_size = min_t(unsigned long, vma->vm_end - vma->vm_start,
                    SZ_4M);
        mutex_unlock(&binder_alloc_mmap_lock);
// Note: 将Virtual Memory Area的起始地址保存在binder_alloc.buffer中
        alloc->buffer = (void __user *)vma->vm_start;
// Note: 
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

&emsp;[ProcessState::setThreadPoolMaxThreadCount][setThreadPoolMaxThreadCountLink]  
&emsp;&emsp;[ioctl(mDriverFD, BINDER_SET_MAX_THREADS, &maxThreads)][ioctl_BINDER_SET_MAX_THREADS_lk]  
&emsp;[ProcessState::setCallRestriction][PSsetCallRestrictionLink]  

[setThreadPoolMaxThreadCountLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=346
[ioctl_BINDER_SET_MAX_THREADS_lk]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=350
[PSsetCallRestrictionLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=227

## 三、实例化ServiceManager类并成为Binder IPC的上下文管理者

&emsp;[ServiceManager::ServiceManager][ServiceManagerCnstctrLink]  
&emsp;[ServiceManager::addService][ServiceManagerAddServiceLink]  
&emsp;[IPCThreadState::setTheContextObject][IPCTSsetTheContextObjectLink]  
&emsp;[ProcessState::becomeContextManager][PSbecomeContextManagerLink]  
&emsp;&emsp;[ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR_EXT, &obj)][ioctl_BINDER_SET_CONTEXT_MGR_EXT_lk]  

[ServiceManagerCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/ServiceManager.cpp;l=119
[ServiceManagerAddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/ServiceManager.cpp;l=213
[IPCTSsetTheContextObjectLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=1110
[PSbecomeContextManagerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=147
[ioctl_BINDER_SET_CONTEXT_MGR_EXT_lk]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=147

## 四、进入无限循环

&emsp;[Looper::prepare][LooperPrepareLink]  
&emsp;&emsp;[Looper::Looper][LooperLooperLink]  

[LooperPrepareLink]:https://cs.android.com/android/platform/superproject/+/master:system/core/libutils/Looper.cpp;l=113
[LooperLooperLink]:https://cs.android.com/android/platform/superproject/+/master:system/core/libutils/Looper.cpp;l=62

设置Looper监听Binder驱动的文件描述符以及事件到来后的回调：  

&emsp;[BinderCallback::setupTo][BinderCallbackSetupToLink]  
&emsp;&emsp;[IPCThreadState::setupPolling][IPCThreadStateSetupPollingLink]  
&emsp;&emsp;&emsp;[IPCThreadState::flushCommands][IPCTSflushCommandsLink]  
&emsp;&emsp;&emsp;&emsp;[IPCThreadState::talkWithDriver][talkWithDriverLink]（BC_ENTER_LOOPER？？？）  

```c++
class BinderCallback : public LooperCallback {
public:
    static sp<BinderCallback> setupTo(const sp<Looper>& looper) {
// ... 省略代码
    }

    int handleEvent(int /* fd */, int /* events */, void* /* data */) override {
        IPCThreadState::self()->handlePolledCommands();
        return 1;  // Continue receiving callbacks.
    }
};
```

[BinderCallbackSetupToLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/main.cpp;l=41
[IPCThreadStateSetupPollingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=647
[IPCTSflushCommandsLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=473
[talkWithDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=965

&emsp;&emsp;[Looper::addFd][LooperaddFdLink]  

[LooperaddFdLink]:https://cs.android.com/android/platform/superproject/+/master:system/core/libutils/Looper.cpp;l=430

设置Looper监听timer fd，并设置相应的回调：  

&emsp;[ClientCallbackCallback::setupTo][ClientCallbackCallbackSetupToLink]  

[ClientCallbackCallbackSetupToLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/main.cpp;l=67

```c++
// LooperCallback for IClientCallback
class ClientCallbackCallback : public LooperCallback {
public:
    static sp<ClientCallbackCallback> setupTo(const sp<Looper>& looper, const sp<ServiceManager>& manager) {
// ... 省略代码
    }

    int handleEvent(int fd, int /*events*/, void* /*data*/) override {
        uint64_t expirations;
        int ret = read(fd, &expirations, sizeof(expirations));
        if (ret != sizeof(expirations)) {
            ALOGE("Read failed to callback FD: ret: %d err: %d", ret, errno);
        }

        mManager->handleClientCallbacks();
        return 1;  // Continue receiving callbacks.
    }
// ... 省略代码
};
```

最后进入无限循环：  

```c++
    while(true) {
        looper->pollAll(-1);
    }
```

参考资料：[《Android Binder框架实现之servicemanager守护进程》](https://blog.csdn.net/tkwxty/article/details/102904305)
