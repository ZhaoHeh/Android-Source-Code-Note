# 一、System Service（server端）注册service的过程

## AMS（ActivityManagerService）注册过程分析

[ActivityManagerService#setSystemProcess][AMSsetSystemProcessLink]  
&emsp;[ServiceManager#addService][SvcMgraddServiceLink]  
&emsp;&emsp;[ServiceManagerProxy#addService][SMPaddServiceLink]  
&emsp;&emsp;&emsp;[IServiceManager.Stub.Proxy#addService][ISMSPaddServiceLink]  
&emsp;&emsp;&emsp;&emsp;[IServiceManager.Stub.Proxy#addService][ISMSPaddServiceLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel#writeStrongBinder][ParcelwriteStrongBinderLink]
&emsp;&emsp;&emsp;&emsp;&emsp;[BinderProxy#transact][BinderProxytransactLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[native BinderProxy#transactNative][BinderProxytransactNativeLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[BpBinder::transact][BpBindertransactLink]  

[AMSsetSystemProcessLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=2102
[SvcMgraddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManager.java;l=179
[SMPaddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/ServiceManagerNative.java;l=70
[ISMSPaddServiceLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=414
[ParcelwriteStrongBinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=844
[BinderProxytransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/BinderProxy.java;l=495
[BinderProxytransactNativeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=1376
[BpBindertransactLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/BpBinder.cpp;l=213

记录：  
com.android.server.am.ActivityManagerService是一个 android.os.IBinder ， 其父类型为 IActivityManager.Stub ；  

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
