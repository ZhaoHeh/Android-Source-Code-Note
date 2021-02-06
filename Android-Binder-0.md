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
[/frameworks/native/cmds/servicemanager/](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/)  
[/frameworks/native/cmds/servicemanager/main.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/cmds/servicemanager/main.cpp;l=123)

**binder driver（不属于AOSP，厂商源码的kernel路径下一定会找到，最新的Linux源码也已经合入，网上也可以很方便地找到）:**  
[binder.c](https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c)  
[binder.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/android/binder.h)  

## 列举一些问题

1. Binder IPC机制的一次拷贝是如何实现的？

## 划分一些部分

1. ServiceManager的初始化

2. add service

3. get service

4. service manager进程

5. startActivity实例
