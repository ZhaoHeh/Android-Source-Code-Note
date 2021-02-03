# 一、System Service（server端）注册service的过程

## AMS（ActivityManagerService）注册过程分析

[ActivityManagerService#setSystemProcess][AMSsetSystemProcessLink]  
&emsp;[ServiceManager#addService][SvcMgraddServiceLink]  

[AMSsetSystemProcessLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=2102
[SvcMgraddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=179

在第一篇已经分析过ServiceManager的实际组成，这里不在赘述，下面主要分析Stub类型的system service作为一个参数的加工过程：  

&emsp;&emsp;[ServiceManagerProxy#addService][SMPaddServiceLink]  
&emsp;&emsp;&emsp;[IServiceManager.Stub.Proxy#addService][ISMSPaddServiceLink]  
&emsp;&emsp;&emsp;&emsp;[Parcel#writeStrongBinder][ParcelwriteStrongBinderLink]（这里IBinder类型的参数，实际类型为ActivityManagerService，父类型为IActivityManager.Stub）  
&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel#nativeWriteStrongBinder][nativeWriteStrongBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[ibinderForJavaObject][ibinderForJavaObjectLink]  

```c++
sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;

    // Instance of Binder?
    if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env->GetLongField(obj, gBinderOffsets.mObject);
        return jbh->get(env, obj);
    }

    // Instance of BinderProxy?
    if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        return getBPNativeData(env, obj)->mObject;
    }

    ALOGW("ibinderForJavaObject: %p is not a Binder object", obj);
    return NULL;
}
```

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[JavaBBinderHolder::get][JavaBBinderHoldergetLink]（这里返回的是一个JavaBBinder对象，继承自BBinder）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::writeStrongBinder][CppParcelWriteStrongBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::flattenBinder][ParcelFlattenBinderLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::writeObject][CppParcelWriteObjectLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::finishWrite][CppParcelFinishWriteLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::finishFlattenBinder][CppParcelFinishFlattenBinderLink]  

[SMPaddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=70
[ISMSPaddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=414
[ParcelwriteStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=844
[nativeWriteStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_os_Parcel.cpp;l=327
[ibinderForJavaObjectLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=783
[JavaBBinderHoldergetLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=458
[CppParcelWriteStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=1136
[ParcelFlattenBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=192
[CppParcelWriteObjectLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=1376
[CppParcelFinishWriteLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=642
[CppParcelFinishFlattenBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=167

总结：  

&emsp;&emsp;&emsp;&emsp;[BinderProxy#transact][BinderProxytransactLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[**native BinderProxy#transactNative**][BinderProxytransactNativeLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[BpBinder::transact][BpBindertransactLink]  

[BinderProxytransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/BinderProxy.java;l=495
[BinderProxytransactNativeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=1376
[BpBindertransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp;l=213

[BpBinder::transact][BpBindertransactLink]  
&emsp;[IPCThreadState::transact][IPCThreadStatetransactLink]  
&emsp;&emsp;[IPCThreadState::writeTransactionData][writeTransactionDataLink]（打包要发送给binder驱动的数据）  
&emsp;&emsp;[IPCThreadState::waitForResponse][waitForResponseLink]  
&emsp;&emsp;&emsp;[IPCThreadState::talkWithDriver][talkWithDriverLink]  
&emsp;&emsp;&emsp;&emsp;[IPCThreadState::talkWithDriver][talkWithDriverLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)][IOCTLSystemCallDriverLink]  

[BpBindertransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp;l=213
[IPCThreadStatetransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=682
[writeTransactionDataLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=1072
[waitForResponseLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=870
[talkWithDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=965
[IOCTLSystemCallDriverLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/IPCThreadState.cpp;l=1018

下面进入驱动执行：  

[binder_ioctl][binder_ioctl_lk]  
&emsp;[binder_ioctl_write_read][binder_ioctl_write_read_lk]  
&emsp;&emsp;[binder_thread_write][binder_thread_write_lk]  
&emsp;&emsp;&emsp;[binder_transaction][binder_transaction_lk]  
&emsp;&emsp;&emsp;&emsp;[wake_up_interruptible???][wake_up_interruptible_lk]  
&emsp;&emsp;[binder_thread_read][binder_thread_read_lk]  

[binder_ioctl_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L4999
[binder_ioctl_write_read_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L4824
[binder_thread_write_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L3577
[binder_transaction_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L2814
[wake_up_interruptible_lk]:https://blog.csdn.net/prike/article/details/76609821
[binder_thread_read_lk]:https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c#L4162

## 附

参考资料：[《Android Binder框架实现之Native层addService详解之请求的发送》](https://blog.csdn.net/tkwxty/article/details/103243685)
