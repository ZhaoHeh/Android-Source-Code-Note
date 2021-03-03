# 每一个Zygote孵化的进程，都已经自动开启了Binder线程池

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