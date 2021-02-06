# Zygote进程中android.os.ServiceManager类的初始化

本篇需要阐述清楚的问题：  

1. android.os.ServiceManager#sServiceManager 是一个 android.os.IServiceManager ，实际类型为 android.os.ServiceManagerProxy ；

2. android.os.ServiceManagerProxy#mServiceManager 也是一个 android.os.IServiceManager ，实际类型为 android.os.IServiceManager.Stub.Proxy ；

3. android.os.IServiceManager.Stub.Proxy#mRemote 是一个 android.os.IBinder ，实际类型为 android.os.BinderProxy ；

4. android.os.BinderProxy#mNativeData 指向的是native类 android::BinderProxyNativeData 的对象， 且此对象的mObject字段实际类型为 android::BpBinder ；

5. android::BpBinder::mHandle 是 ServiceManager 的句柄，值为0 ；

6. Stub.asInterface函数的作用：同一进程直接返回server，不是同一进程返回一个代理作为client端；

[static ServiceManager#getService][SvcMgrGetServiceLink]  
&emsp;[static ServiceManager#rawGetService][SvcMgrRawGetServiceLink]  
&emsp;&emsp;[static ServiceManager#getIServiceManager][SvcMgrGetIServiceManagerLink]  

[SvcMgrGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=128
[SvcMgrRawGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=318
[SvcMgrGetIServiceManagerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=110

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

在应用进程中，service manager的客户端代理就是android.os.ServiceManager#sServiceManager引用的android.os.IServiceManager对象。  

&emsp;&emsp;&emsp;[static native BinderInternal#getContextObject][BinderInternalGetCtxObjLink]  

[BinderInternalGetCtxObjLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=1130

```c++
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
// 代码路径：frameworks/base/core/jni/android_util_Binder.cpp
```

&emsp;&emsp;&emsp;&emsp;[ProcessState::getContextObject][ProcessStateGetCtxObjLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[ProcessState::getStrongProxyForHandle][ProcessStateGSPFHLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[ProcessState::lookupHandleLocked][PSLookupHandleLockedLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[IPCThreadState::transact][IPCTransactLink]（IBinder::PING_TRANSACTION）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[BpBinder::create][BpBinderCreateLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[BpBinder::BpBinder][BpBinderBpBinderLink]  

[ProcessStateGetCtxObjLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=123
[ProcessStateGSPFHLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=247
[PSLookupHandleLockedLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=234
[IPCTransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=234
[BpBinderCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp;l=110
[BpBinderBpBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp;l=139

现在，应用进程中new了一个android::BpBinder对象，成员变量android::BpBinder::mHandle的值是0，这个mHandle实际上执行的就是Binder驱动中service manager相关的Binder实体。  

&emsp;&emsp;&emsp;&emsp;[javaObjectForIBinder][javaObjectForIBinderLink]（将一个android::IBinder封装为一个android.os.BinderProxy类型）  
&emsp;&emsp;&emsp;&emsp;&emsp;[BinderProxyNativeData][BinderProxyNativeDataLink]（将BpBinder对象封装进结构BinderProxyNativeData中）  
&emsp;&emsp;&emsp;&emsp;&emsp;[BinderProxy#getInstance][BinderProxyGetInstanceLink]（将BinderProxyNativeData对象封装进Java对象BinderProxy中）  

[javaObjectForIBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=739
[BinderProxyNativeDataLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=720
[BinderProxyGetInstanceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=757

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
