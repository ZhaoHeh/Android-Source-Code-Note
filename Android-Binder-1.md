# Zygote进程中android.os.ServiceManager类的初始化

本篇需要阐述清楚的问题：  

1. **android.os.ServiceManager#sServiceManager**是一个**android.os.IServiceManager**，实际类型为**android.os.ServiceManagerProxy**；

2. **android.os.ServiceManagerProxy#mServiceManager**也是一个**android.os.IServiceManager**，实际类型为**android.os.IServiceManager.Stub.Proxy**；

3. **android.os.IServiceManager.Stub.Proxy#mRemote**是一个 android.os.IBinder*，实际类型为**android.os.BinderProxy**；

4. **android.os.BinderProxy#mNativeData**指向的是native类**android::BinderProxyNativeData**的对象，且此对象的**mObject**字段实际类型为**android::BpBinder**；

5. **android::BpBinder::mHandle**是**service manager**的binder实体在驱动中的句柄，值为0；

6. Stub.asInterface函数的作用：同一进程直接返回server，不是同一进程返回一个代理作为client端。

## 一、Java层

我们从Activity#startActivity()的开始：  

```java
    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
```

[Activity#startActivity][startActivityLink1]  
&emsp;[Activity#startActivity][startActivityLink2]  
&emsp;&emsp;[Activity#startActivityForResult][startActivityLink3]  
&emsp;&emsp;&emsp;[Activity#execStartActivity][execStartActivityLink]  

[startActivityLink1]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/Activity.java;l=5616
[startActivityLink2]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/Activity.java;l=5643
[startActivityLink3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/Activity.java;l=5315
[execStartActivityLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/Instrumentation.java;l=1693

```java
    @UnsupportedAppUsage
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        // ... 省略代码
        try {
            intent.migrateExtraStreamToClipData(who);
            intent.prepareToLeaveProcess(who);
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getBasePackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

在startActivity之前，需要先通过**ActivityTaskManager#getService**获取ActivityManagerService的客户端，我们这里从startActivity的调用转向，查看**ActivityTaskManager#getService**的调用：  

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
```

[ActivityTaskManager#getService][ATMgetServiceLink]  
&emsp;[Singleton#get][SingletonGet]  
&emsp;&emsp;[Singleton#create][SingletonCreate]  
&emsp;&emsp;&emsp;[ServiceManager#getService][SMgetManager]  

[ATMgetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityTaskManager.java;l=149
[SingletonGet]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/util/Singleton.java;l=40
[SingletonCreate]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityTaskManager.java;l=157
[SMgetManager]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=128

```java
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return Binder.allowBlocking(rawGetService(name));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }
```

[static ServiceManager#getService][SvcMgrGetServiceLink]  
&emsp;[static ServiceManager#rawGetService][SvcMgrRawGetServiceLink]  

[SvcMgrGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=128
[SvcMgrRawGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=318

```java
    private static IBinder rawGetService(String name) throws RemoteException {
        final long start = sStatLogger.getTime();

        final IBinder binder = getIServiceManager().getService(name);
        // ... 省略代码
    }
```

在通过getService获取ActivityManagerService的客户端之前，需要先通过**ServiceManager#getIServiceManager**获取**service manager**的客户端，我们这里再一次转向，查看**ServiceManager#getIServiceManager**的调用：  

```java
    @UnsupportedAppUsage
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }
// 代码路径：frameworks/base/core/java/android/os/ServiceManager.java
```

[static ServiceManager#getIServiceManager][SvcMgrGetIServiceManagerLink]  
&emsp;[static native BinderInternal#getContextObject][BinderInternalGetCtxObjLink]  

[SvcMgrGetIServiceManagerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=110
[BinderInternalGetCtxObjLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=1130

native方法**BinderInternal#getContextObject**是专门用来获取service manager客户端在native层对象的方法，从名字就可以看出来，context object意为上下文对象，实际上和服务端的native对象被叫做上下文管理者对应。  

## 二、native层

### jni

[static native BinderInternal#getContextObject][BinderInternalGetCtxObjLink]  

```c++
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
// 代码路径：frameworks/base/core/jni/android_util_Binder.cpp
```

### native

&emsp;[ProcessState::getContextObject][ProcessStateGetCtxObjLink]  
&emsp;&emsp;[ProcessState::getStrongProxyForHandle][ProcessStateGSPFHLink]  
&emsp;&emsp;&emsp;[ProcessState::lookupHandleLocked][PSLookupHandleLockedLink]  
&emsp;&emsp;&emsp;[IPCThreadState::transact][IPCTransactLink]（IBinder::PING_TRANSACTION）  
&emsp;&emsp;&emsp;[BpBinder::create][BpBinderCreateLink]  
&emsp;&emsp;&emsp;&emsp;[BpBinder::BpBinder][BpBinderBpBinderLink]  

[ProcessStateGetCtxObjLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=123
[ProcessStateGSPFHLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=247
[PSLookupHandleLockedLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=234
[IPCTransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=234
[BpBinderCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp;l=110
[BpBinderBpBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp;l=139

现在，应用进程中new了一个android::BpBinder对象，成员变量android::BpBinder::mHandle的值是0，这个mHandle实际上执行的就是Binder驱动中service manager相关的Binder实体。  

### 回到jni

&emsp;[javaObjectForIBinder][javaObjectForIBinderLink]（将一个android::IBinder封装为一个android.os.BinderProxy类型）  
&emsp;&emsp;[BinderProxyNativeData][BinderProxyNativeDataLink]（将BpBinder对象封装进结构BinderProxyNativeData中）  
&emsp;&emsp;[BinderProxy#getInstance][BinderProxyGetInstanceLink]（将BinderProxyNativeData对象封装进Java对象BinderProxy中）  

[javaObjectForIBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=739
[BinderProxyNativeDataLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=720
[BinderProxyGetInstanceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=757

## 三、回到Java层

&emsp;&emsp;&emsp;[static Binder#allowBlocking][BinderAllowBlockingLink]  
&emsp;&emsp;&emsp;[static ServiceManagerNative#asInterface][SvcMgrNtvAsInterfaceLink]  
&emsp;&emsp;&emsp;&emsp;[ServiceManagerProxy#ServiceManagerProxy][ServiceManagerProxyLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[IServiceManager.Stub#asInterface][StubAsInterfaceLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[IServiceManager.Stub.Proxy#Proxy][ProxyConstructorLink]  

```java
    public static android.os.IServiceManager asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof android.os.IServiceManager))) {
        return ((android.os.IServiceManager)iin);
      }
      return new android.os.IServiceManager.Stub.Proxy(obj);
    }
    // ... 省略代码
    private static class Proxy implements android.os.IServiceManager
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
        mRemote = remote;
      }
    // ... 省略代码

// 代码路径：out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java
```

[BinderAllowBlockingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Binder.java;l=213
[SvcMgrNtvAsInterfaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=38
[ServiceManagerProxyLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=51
[StubAsInterfaceLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=122
[ProxyConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=335

到此为止，对service manager这一依赖Binder的system service的客户端的封装完成，下面我们来重复以下开头的结论：  

（1）**android.os.ServiceManager#sServiceManager**是一个**android.os.IServiceManager**，实际类型为**android.os.ServiceManagerProxy**；  
（2）**android.os.ServiceManagerProxy#mServiceManager**也是一个**android.os.IServiceManager**，实际类型为**android.os.IServiceManager.Stub.Proxy**；  
（3）**android.os.IServiceManager.Stub.Proxy#mRemote**是一个**android.os.IBinder**，实际类型为**android.os.BinderProxy**；  
（4）**android.os.BinderProxy#mNativeData**指向的是native类型**android::BinderProxyNativeData**的对象，**android::BinderProxyNativeData::mObject**指向的是**android::BpBinder**的对象；  
（5）上述此BpBinder对象中的**android::BpBinder::mHandle**成员变量是server manager的Binder实体在Binder驱动中的标识符，值为固定的0；
