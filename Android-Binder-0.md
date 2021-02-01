# zygote进程中android.os.ServiceManager类的初始化

本篇需要阐述清楚的问题：  

0. Stub.asInterface函数的作用；

1. android.os.ServiceManager#sServiceManager 是一个 android.os.IServiceManager ，实际类型为 android.os.ServiceManagerProxy ；

2. android.os.ServiceManagerProxy#mServiceManager 也是一个 android.os.IServiceManager ，实际类型为 android.os.IServiceManager.Stub.Proxy ；

3. android.os.IServiceManager.Stub.Proxy#mRemote 是一个 android.os.IBinder ，实际类型为 android.os.BinderProxy ；

4. android.os.BinderProxy#mNativeData 指向的是native类 android::BinderProxyNativeData 的对象， 且此对象的mObject字段实际类型为 android::BpBinder ；

5. android::BpBinder::mHandle 是 ServiceManager 的句柄，值为0 ？？？ ；
