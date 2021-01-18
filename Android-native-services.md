# 模块信息

## AudioServer

### 1.1 [frameworks/av/media/audioserver/main_audioserver.cpp][link3]  

[link3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/av/media/audioserver/main_audioserver.cpp

```c++
int main(int argc __unused, char **argv)
{
    // ... 省略代码
    pid_t childPid;
    // FIXME The advantage of making the process containing media.log service the parent process of
    // the process that contains the other audio services, is that it allows us to collect more
    // detailed information such as signal numbers, stop and continue, resource usage, etc.
    // But it is also more complex.  Consider replacing this by independent processes, and using
    // binder on death notification instead.
    if (doLog && (childPid = fork()) != 0) {
        // media.log service
        //prctl(PR_SET_NAME, (unsigned long) "media.log", 0, 0, 0);
        // unfortunately ps ignores PR_SET_NAME for the main thread, so use this ugly hack
        strcpy(argv[0], "media.log");
        sp<ProcessState> proc(ProcessState::self());
        MediaLogService::instantiate();
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
        for (;;) {
            // ... 省略代码
        }
    } else {
        // all other services
        if (doLog) {
            prctl(PR_SET_PDEATHSIG, SIGKILL);   // if parent media.log dies before me, kill me also
            setpgid(0, 0);                      // but if I die first, don't kill my parent
        }
        android::hardware::configureRpcThreadpool(4, false /*callerWillJoin*/);
        sp<ProcessState> proc(ProcessState::self());
        sp<IServiceManager> sm = defaultServiceManager();
        ALOGI("ServiceManager: %p", sm.get());
        AudioFlinger::instantiate();
        AudioPolicyService::instantiate();

        // AAudioService should only be used in OC-MR1 and later.
        // And only enable the AAudioService if the system MMAP policy explicitly allows it.
        // This prevents a client from misusing AAudioService when it is not supported.
        aaudio_policy_t mmapPolicy = property_get_int32(AAUDIO_PROP_MMAP_POLICY,
                                                        AAUDIO_POLICY_NEVER);
        if (mmapPolicy == AAUDIO_POLICY_AUTO || mmapPolicy == AAUDIO_POLICY_ALWAYS) {
            AAudioService::instantiate();
        }

        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
    }
}
```

### 1.2 [frameworks/av/media/audioserver/audioserver.rc][link2]

[link2]:https://cs.android.com/android/platform/superproject/+/master:frameworks/av/media/audioserver/audioserver.rc

```rc
service audioserver /system/bin/audioserver
    class core
    user audioserver
    # media gid needed for /dev/fm (radio) and for /data/misc/media (tee)
    group audio camera drmrpc media mediadrm net_bt net_bt_admin net_bw_acct wakelock
    capabilities BLOCK_SUSPEND
    ioprio rt 4
    task_profiles ProcessCapacityHigh HighPerformance
    onrestart restart vendor.audio-hal
    onrestart restart vendor.audio-hal-4-0-msd
    # Keep the original service names for backward compatibility
    onrestart restart vendor.audio-hal-2-0
    onrestart restart audio-hal-2-0

on property:vts.native_server.on=1
    stop audioserver
on property:vts.native_server.on=0
    start audioserver
```

### 1.3 [frameworks/av/media/audioserver/Android.mk][link1]  

[link1]:https://cs.android.com/android/platform/superproject/+/master:frameworks/av/media/audioserver/Android.mk

```make
LOCAL_MODULE := audioserver

LOCAL_INIT_RC := audioserver.rc

LOCAL_CFLAGS := -Werror -Wall

include $(BUILD_EXECUTABLE)
```

### 1.4 正在运行的手机adb shell中执行

```shell
PAFM00:/system/bin # ls -al | grep audioserver
-rwxr-xr-x  1 root   shell      20488 2019-05-23 10:23 audioserver

PAFM00:/system/bin # ps -Al
F   S       UID   PID       PPID    C       PRI     NI      BIT     SZ      WCHAN               TTY             TIME            CMD
4   S       1041  1257      1       0       19      0       32      25813   binder_thread_read  ?               00:00:01        audioserver
```

## SurfaceFlinger

### 2.1 SurfaceFlinger源码路径

frameworks/native/services/surfaceflinger/  
frameworks/native/services/surfaceflinger/Android.bp  
frameworks/native/services/surfaceflinger/surfaceflinger.rc  
frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp  

### 2.4 正在运行的手机adb shell中执行

```shell
PAFM00:/system/bin # ls -al
-rwxr-xr-x  1 system graphics   37096 2019-05-23 10:23 surfaceflinger

PAFM00:/system/bin # ps -Al
USER           PID  PPID     VSZ    RSS WCHAN            ADDR FRZ  S NAME                       
root             1     0   21368   4544 SyS_epoll_wait      0 efg  S init
system         770     1 2250228  53240 SyS_epoll_wait      0 ebg  S surfaceflinger
```
