# Binder

## 代码路径

**java binder:**  
/frameworks/base/core/java/android/os/Binder.java  
/frameworks/base/core/java/android/os/BinderProxy.java  
/frameworks/base/core/java/android/os/IBinder.java  
/frameworks/base/core/java/android/os/ServiceManager.java  

**native binder:**  
/frameworks/native/libs/binder/

**native service manager:**  
/frameworks/native/cmds/servicemanager/  

**binder driver（不属于AOSP，厂商源码的kernel路径下一定会找到，最新的Linux源码也已经合入，网上也可以很方便地找到）:**  
[binder.c](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c)  
[binder.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/android/binder.h)  

## 从一个简单例子开始

在某个线程中调用:  

[Activity#startActivity][ActivityStartActivityLink]  
&emsp;[Activity#startActivityForResult][startActivityForResultLink]  
&emsp;&emsp;[Instrumentation#execStartActivity][execStartActivityLink]  

[ActivityStartActivityLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/Activity.java;l=5643
[startActivityForResultLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/Activity.java;l=5277
[execStartActivityLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/Instrumentation.java;l=1693

```java
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getBasePackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
// 代码路径：frameworks/base/core/java/android/app/Instrumentation.java
```

&emsp;&emsp;&emsp;[ActivityTaskManager#getService][ATMarGetServiceLink]  

[ATMarGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityTaskManager.java;l=149

```java
    /** @hide */
    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }

    @UnsupportedAppUsage(trackingBug = 129726065)
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
// 代码路径：frameworks/base/core/java/android/app/ActivityTaskManager.java
```

[IActivityTaskManager.Stub#asInterface][asInterfaceLink]  

[asInterfaceLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/app/IActivityTaskManager.java;l=719

```java
    /**
     * Cast an IBinder object into an android.app.IActivityTaskManager interface,
     * generating a proxy if needed.
     */
    public static android.app.IActivityTaskManager asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof android.app.IActivityTaskManager))) {
        return ((android.app.IActivityTaskManager)iin);
      }
      return new android.app.IActivityTaskManager.Stub.Proxy(obj);
    }
// ... 省略代码
  public static final java.lang.String DESCRIPTOR = "android.app.IActivityTaskManager";
// ... 省略代码

// 代码路径：无，编译过程中生成的
```

下面下两个没有证据支持的断言：  

1. Binder IPC过程中, 同一进程的调用，asInterface()方法返回的是本地Binder对象，不同进程的调用，则返回远程代理对象BinderProxy.  

2. ServiceManager.getService返回的是指向目标服务的代理对象--BinderProxy对象，由该代理对象可以找到目标服务所在进程

[IActivityTaskManager.Stub.Proxy#startActivity][ProxyStartActivityLink]  
&emsp;[BinderProxy#transact][BinderProxyTransactLink]  
&emsp;&emsp;[BinderProxy#transactNative][BinderProxyTransactNativeLink]  
&emsp;&emsp;&emsp;[getBPNativeData][getBPNativeDataLink]  

[ProxyStartActivityLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/app/IActivityTaskManager.java;l=3656
[BinderProxyTransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/BinderProxy.java;l=495
[BinderProxyTransactNativeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=1376
[getBPNativeDataLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=732

```c++
BinderProxyNativeData* getBPNativeData(JNIEnv* env, jobject obj) {
    return (BinderProxyNativeData *) env->GetLongField(obj, gBinderProxyOffsets.mNativeData);
}
```

再下一个断言：gBinderProxyOffsets.mNativeData中保存的是BpBinder对象, 这是开机时Zygote调用AndroidRuntime::startReg方法来完成jni方法的注册.  

&emsp;&emsp;&emsp;[BpBinder::transact][BpBinderTransactLink]  
&emsp;&emsp;&emsp;&emsp;[IPCThreadState::transact][IPCThreadStateTransactLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[IPCThreadState::waitForResponse][IPCThreadStateWaitForResponseLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[IPCThreadState::talkWithDriver][IPCThreadStateTalkWithDriverLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;```ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)```

[BpBinderTransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp;l=213
[IPCThreadStateTransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=682
[IPCThreadStateWaitForResponseLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=870
[IPCThreadStateTalkWithDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=965
