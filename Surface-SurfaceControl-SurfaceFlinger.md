# Android中的Surface

## 1. SurfaceControl.Builder创建SurfaceControl对象

调用栈如下：  
[SurfaceControl.Builder#build][buildLink]  
&emsp;[SurfaceControl#SurfaceControl][ctrlLink]  
&emsp;&emsp;[SurfaceControl#nativeCreate][ctrlNativeLink]  
&emsp;&emsp;&emsp;[SurfaceComposerClient::createSurfaceChecked][createSfLink]  
&emsp;&emsp;&emsp;&emsp;[BpSurfaceComposerClient::createSurface][bpCreateLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[Client::createSurface][bnCreateLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SurfaceFlinger::createLayer][createLayerLink]  

### 1. SurfaceControl对象的创建

[SurfaceControl(java)][ctrlLink](frameworks/base/core/java/android/view/SurfaceControl.java)  
&emsp;&emsp;|  
&emsp;&emsp;|[SurfaceControl][ctrlLink]#mNativeObject  
&emsp;&emsp;|  
[SurfaceControl(c++)][nativeCtrlLink](frameworks/native/libs/gui/SurfaceControl.cpp)  
  
SurfaceControl(c++)的创建由SurfaceComposerClient::createSurfaceChecked完成，对象本身是在local线程中创建的，但是所需要的IBinder和IGraphicBufferProducer信息则需要借助Binder机制由服务端提供。  
SurfaceControl(c++)的构造函数的任务就是给四个成员变量赋值，分别是:  
&emsp;SurfaceControl::mClient -- SurfaceComposerClient类型  
&emsp;SurfaceControl::mHandle -- IBinder>型  
&emsp;SurfaceControl::mGraphicBufferProducer -- IGraphicBufferProducer类型  
&emsp;SurfaceControl::mTransformHint -- uint32_t类型  
  
由此可知，SurfaceControl(c++)对象中保存的核心信息就是**IGraphicBufferProducer对象**SurfaceControl::mGraphicBufferProducer

### 2. 由SurfaceControl创建Surface对象

调用栈如下：  
[Surface#Surface][surfaceconstructorLink]  
&emsp;[Surface#copyFrom][copyFromLink]  
上面的流程就是将创建的Surface(c++)的指针赋值给mNativeObject，完成Surface(c++)对象和Surface(java)对象的绑定  
&emsp;&emsp;[Surface#nativeGetFromSurfaceControl][nativeGetFromSurfaceControlLink]  
&emsp;&emsp;&emsp;[SurfaceControl::getSurface][ctrlgetSfLink]  
&emsp;&emsp;&emsp;&emsp;[SurfaceControl::generateSurfaceLocked][generateLink]  
SurfaceControl(c++)创建Surface(c++)传递的唯一信息就是**IGraphicBufferProducer对象**，当然我们也能发现，由SurfaceControl(c++)创建的Surface(c++)，producer都是不受App自身控制的  
&emsp;&emsp;&emsp;&emsp;&emsp;[Surface::Surface][sfConstructorLink]  

### 3. IGraphicBufferProducer对象 -- SurfaceControl和Surface的本质是producer

从上面的代码可以看出，SurfaceControl和Surface的创建、一个Surface区别于另一个Surface的关键，都在于他们的native对象保存的IGraphicBufferProducer对象，有必要对IGraphicBufferProducer一探究竟。  

1. IGraphicBufferProducer的作用  

2. IGraphicBufferProducer的具体实现

总结：持有IGraphicBufferProducer的surface从buffer queue的角度看就是graphic buffer的生产者。更进一步，持有surface的App就是graphic buffer的生产者。

### 4. Surface和ANativeWindow -- 从OpenGL的角度理解Surface

[ANativeWindow][anwDefineLink]是一个结构体，定义在frameworks/native/libs/nativewindow/include/system/window.h中，这个结构体是中的主要成员变量都是函数指针，如下所示：  

```c++
struct ANativeWindow
{
    int     (*setSwapInterval)(struct ANativeWindow* window,
                int interval);
    int     (*query)(const struct ANativeWindow* window,
                int what, int* value);
    int     (*perform)(struct ANativeWindow* window,
                int operation, ... );
    int     (*dequeueBuffer)(struct ANativeWindow* window,
                struct ANativeWindowBuffer** buffer, int* fenceFd);
    int     (*queueBuffer)(struct ANativeWindow* window,
                struct ANativeWindowBuffer* buffer, int fenceFd);
    int     (*cancelBuffer)(struct ANativeWindow* window,
                struct ANativeWindowBuffer* buffer, int fenceFd);
}
```

所以可以把ANativeWindow理解为一个API，定义了一些有关graphic buffer的操作供OpenGL或Vulkan的实现者调用，而这个API在Android App中的实现者就是[Surface(c++)][nSfHeadLink]类。  
在《》的第二节中我们知道，Android App中的Surface信息会通过  函数告知penGL或Vulkan的实现者，虽然我们不知道penGL或Vulkan的具体实现，但是ANativeWindow的定义决定了OpenGL或Vulkan的实现者可以对这块buffer执行的操作，当penGL或Vulkan的实现者想要操作这块buffer时，我们以ANativeWindow::setSwapInterval为例看看实际上是对Android App中的Surface做了什么：

```c++
Surface::Surface(const sp<IGraphicBufferProducer>& bufferProducer, bool controlledByApp)
      : mGraphicBufferProducer(bufferProducer),
        mCrop(Rect::EMPTY_RECT),
        mBufferAge(0),
        mGenerationNumber(0),
        mSharedBufferMode(false),
        mAutoRefresh(false),
        mSharedBufferSlot(BufferItem::INVALID_BUFFER_SLOT),
        mSharedBufferHasBeenQueued(false),
        mQueriedSupportedTimestamps(false),
        mFrameTimestampsSupportsPresent(false),
        mEnableFrameTimestamps(false),
        mFrameEventHistory(std::make_unique<ProducerFrameEventHistory>()) {
    // Initialize the ANativeWindow function pointers.
    ANativeWindow::setSwapInterval  = hook_setSwapInterval;
    ANativeWindow::dequeueBuffer    = hook_dequeueBuffer;
    ANativeWindow::cancelBuffer     = hook_cancelBuffer;
    ANativeWindow::queueBuffer      = hook_queueBuffer;
    ANativeWindow::query            = hook_query;
    ANativeWindow::perform          = hook_perform;
```

[Surface::hook_setSwapInterval][hook_setSwapIntervalLink]  
[Surface::setSwapInterval][SFsetSwapIntervalLink]  
从中可以看出，实际上还是操作了IGraphicBufferProducer对象。  
总结：OpenGL或Vulkan的实现者持有ANativeWindow，ANativeWindow的实现者Surface(c++)持有IGraphicBufferProducer对象，因此可以说OpenGL或Vulkan的实现者是Android App对应的graphic buffer的生产者。

### 5. TextureView中的Surface

https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureLayer.java;drc=master;l=35
https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureView.java;l=751
https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/SurfaceTexture.java;l=166;drc=master
https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_graphics_SurfaceTexture.cpp;l=253
https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/Surface.java;l=252?q=Surface.java
https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_view_Surface.cpp;l=155;drc=master;bpv=0;bpt=1
https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/Surface.cpp;l=66;drc=master
https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/Surface.java;drc=master;l=680?q=Surface.java

[buildLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/SurfaceControl.java;l=645
[ctrlLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/SurfaceControl.java;l=961
[ctrlNativeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_view_SurfaceControl.cpp;l=227
[createSfLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/SurfaceComposerClient.cpp;l=1638
[bpCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/ISurfaceComposerClient.cpp;l=50
[bnCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/services/surfaceflinger/Client.cpp;l=79
[createLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp;l=3953
[nativeCtrlLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/SurfaceControl.cpp

[surfaceconstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/Surface.java;l=232
[copyFromLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/Surface.java;l=549
[nativeGetFromSurfaceControlLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_view_Surface.cpp;l=282
[ctrlgetSfLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/SurfaceControl.cpp;l=131
[generateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/SurfaceControl.cpp;l=126
[sfConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/Surface.cpp;l=66

[anwDefineLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/nativewindow/include/system/window.h;l=341
[nSfHeadLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/include/gui/Surface.h;l=68
[hook_setSwapIntervalLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/Surface.cpp;l=374
[SFsetSwapIntervalLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/Surface.cpp;l=521
