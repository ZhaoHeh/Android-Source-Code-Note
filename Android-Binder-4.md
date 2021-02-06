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

```c++
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }
```

[PSinitWithDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=80
[PSinitLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=90
[PSConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=395
[PSopen_driverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=367
[mmapCalledLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=395

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
