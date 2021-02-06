# 二、应用进程（client端）获取service的过程

## 1. 在App中startActivity

[Activity#startActivityForResult][ActivityStartActivityForResultLink]  
&emsp;[Instrumentation#execStartActivity][execStartActivityLink]  
&emsp;&emsp;[ActivityTaskManager#getService][ATManagerGetServiceLink]  

[ActivityStartActivityForResultLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/Activity.java;l=5315
[execStartActivityLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/Instrumentation.java;l=1693
[ATManagerGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityTaskManager.java;l=149

```java
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

## 2. ServiceManager#sServiceManager执行getService

&emsp;&emsp;[ServiceManagerProxy#getService][SvcMgrPxyGetServiceLink]  
&emsp;&emsp;&emsp;[IServiceManager.Stub.Proxy#checkService][ProxyCheckServiceLink]  

[SvcMgrPxyGetServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=61
[ProxyCheckServiceLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=387

```java
    @UnsupportedAppUsage
    public IBinder getService(String name) throws RemoteException {
        // Same as checkService (old versions of servicemanager had both methods).
        return mServiceManager.checkService(name);
    }
// 代码路径：frameworks/base/core/java/android/os/ServiceManagerNative.java

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
// 代码路径：out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java
```

## 3. 从BinderProxy#transact进入驱动

？？？

## 4. Parcel#readStrongBinder

&emsp;&emsp;&emsp;&emsp;[Parcel#readStrongBinder][ParcelReadStrongBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[static native Parcel#nativeReadStrongBinder][nativeReadStrongBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::readStrongBinder][CppParcelReadStrongBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::readNullableStrongBinder][readNullableStrongBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::unflattenBinder][ParcelUnflattenBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::readObject][ParcelReadObjectLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[javaObjectForIBinder][javaObjectForIBinderLink]（将一个android::IBinder封装为一个android.os.BinderProxy类型）  

[ParcelReadStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=2493
[nativeReadStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_os_Parcel.cpp;l=504
[CppParcelReadStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=2191
[readNullableStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=2186
[ParcelUnflattenBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=246
[ParcelReadObjectLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=2430
[javaObjectForIBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=739

```c++
status_t Parcel::unflattenBinder(sp<IBinder>* out) const
{
    const flat_binder_object* flat = readObject(false);

    if (flat) {
        switch (flat->hdr.type) {
            case BINDER_TYPE_BINDER: {
                sp<IBinder> binder = reinterpret_cast<IBinder*>(flat->cookie);
                return finishUnflattenBinder(binder, out);
            }
            case BINDER_TYPE_HANDLE: {
                sp<IBinder> binder =
                    ProcessState::self()->getStrongProxyForHandle(flat->handle);
                return finishUnflattenBinder(binder, out);
            }
        }
    }
    return BAD_TYPE;
}
```

## [IActivityTaskManager.Stub.asInterface](out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/app/IActivityTaskManager.java)

```java
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
// 代码路径：out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/app/IActivityTaskManager.java
```

## 应用进程的native层如何获取service

## 附

参考链接：[《Android Binder框架实现之Java层获取Binder服务源码分析》](https://blog.csdn.net/tkwxty/article/details/108165937)

[AMSBindAppLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=5307
[AppBindSelfLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ActivityThread.java;l=1037
