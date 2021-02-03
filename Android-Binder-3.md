# 二、应用进程（client端）获取service的过程

## 应用进程的Java层如何获取service

[static ServiceManager#getService][SvcMgrGetServiceLink]  

```java
    @UnsupportedAppUsage
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

&emsp;[static ServiceManager#rawGetService][SvcMgrRawGetServiceLink]  

[SvcMgrGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=128
[SvcMgrRawGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=318

&emsp;&emsp;[static ServiceManager#getIServiceManager][SvcMgrGetIServiceManagerLink]  

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

&emsp;&emsp;&emsp;[static native BinderInternal#getContextObject][BinderInternalGetCtxObjLink]  
&emsp;&emsp;&emsp;&emsp;[ProcessState::getContextObject][ProcessStateGetCtxObjLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[ProcessState::getStrongProxyForHandle][ProcessStateGSPFHLink]  
&emsp;&emsp;&emsp;&emsp;[javaObjectForIBinder][javaObjectForIBinderLink]（将一个android::IBinder封装为一个android.os.BinderProxy类型）  
&emsp;&emsp;&emsp;[static Binder#allowBlocking][BinderAllowBlockingLink]  
&emsp;&emsp;&emsp;[static ServiceManagerNative#asInterface][SvcMgrNtvAsInterfaceLink]  
&emsp;&emsp;&emsp;&emsp;[ServiceManagerProxy#ServiceManagerProxy][ServiceManagerProxyLink]  

[BinderInternalGetCtxObjLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=1130
[ProcessStateGetCtxObjLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=123
[ProcessStateGSPFHLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/ProcessState.cpp;l=247
[javaObjectForIBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=739
[BinderAllowBlockingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Binder.java;l=213
[SvcMgrNtvAsInterfaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=38
[ServiceManagerProxyLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=51

android.os.ServiceManager#sServiceManager 是一个 android.os.IServiceManager ，实际类型为 android.os.ServiceManagerProxy ；  
android.os.ServiceManagerProxy#mServiceManager 也是一个 android.os.IServiceManager ，实际类型为 android.os.IServiceManager.Stub.Proxy ；  
android.os.IServiceManager.Stub.Proxy#mRemote 是一个 android.os.IBinder ，实际类型为 android.os.BinderProxy ；  

&emsp;&emsp;[ServiceManagerProxy#getService][SvcMgrPxyGetServiceLink]  
&emsp;&emsp;&emsp;[IServiceManager.Stub.Proxy#checkService][ProxyCheckServiceLink]  

```java
      @Override public android.os.IBinder checkService(java.lang.String name) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        android.os.IBinder _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeString(name);
          boolean _status = mRemote.transact(Stub.TRANSACTION_checkService, _data, _reply, 0);
          if (!_status) {
            if (getDefaultImpl() != null) {
              return getDefaultImpl().checkService(name);
            }
          }
          _reply.readException();
          _result = _reply.readStrongBinder();
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }
```

&emsp;&emsp;&emsp;&emsp;[Parcel#readStrongBinder][ParcelReadStrongBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[static native Parcel#nativeReadStrongBinder][nativeReadStrongBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[javaObjectForIBinder][javaObjectForIBinderLink]（将一个android::IBinder封装为一个android.os.BinderProxy类型）  

[SvcMgrPxyGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=61
[ProxyCheckServiceLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=387
[ParcelReadStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=2493
[nativeReadStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_os_Parcel.cpp;l=504
[javaObjectForIBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=739

[AMSBindAppLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=5307
[AppBindSelfLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityThread.java;l=1037

## 应用进程的native层如何获取service

## 附

参考链接：[《Android Binder框架实现之Java层获取Binder服务源码分析》](https://blog.csdn.net/tkwxty/article/details/108165937)
