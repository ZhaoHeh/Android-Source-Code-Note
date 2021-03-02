# Android Binder 学习笔记大纲

## 代码路径

### **java binder**

[/frameworks/base/core/java/android/os/](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/)  
  
例：  
/frameworks/base/core/java/android/os/Binder.java  
/frameworks/base/core/java/android/os/BinderProxy.java  
/frameworks/base/core/java/android/os/IBinder.java  
/frameworks/base/core/java/android/os/ServiceManager.java  

### **jni binder**

[/frameworks/base/core/jni/](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/)  

例：  
/frameworks/base/core/jni/android_os_Parcel.cpp  
/frameworks/base/core/jni/android_util_Binder.cpp  

### **native binder**

[/frameworks/native/libs/binder/](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/)  
  
例：  
/frameworks/native/libs/binder/Parcel.cpp  
/frameworks/native/libs/binder/Binder.cpp（BBinder也在此文件内）  
/frameworks/native/libs/binder/BpBinder.cpp  

### **native service manager**

[/frameworks/native/cmds/servicemanager/](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/)  
  
例：  
/frameworks/native/cmds/servicemanager/main.cpp  
/frameworks/native/cmds/servicemanager/ServiceManager.cpp  

### **binder driver**

Binder驱动不属于AOSP，但在厂商源码的kernel路径下一定会找到，最新的Linux源码也已经合入，网上也可以很方便地搜到，这里就不给出其在Android源码中的路径了  

例：  
[drivers/android/binder.c](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c)  
[include/uapi/linux/android/binder.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/android/binder.h)  

## 将学习笔记划分为如下部分

1. [ServiceManager的初始化](./Android-Binder-1.md)

2. [add service](./Android-Binder-2.md)

3. [get service](./Android-Binder-3.md)

4. [service manager进程](./Android-Binder-4.md)

5. [startActivity实例](./Android-Binder-5.md)

## 带着一些线索学习Binder

1. 一个问题：Binder IPC机制所谓一次拷贝是如何实现的？

2. 一张图片：Android Binder的实现架构：

![Android Binder框架图](./Android-Binder-Refs/Android_Binder_Frame.png "Android Binder框架图")

参考链接：[*Android Binder IPC Framework | Scatter-Gather Optimization*](https://www.pathpartnertech.com/android-binder-ipc-framework-scatter-gather-optimization/)
