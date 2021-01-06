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

```java
    @Override
    public final void draw(Canvas canvas) {
        // NOTE: Maintain this carefully (see View#draw)
        mPrivateFlags = (mPrivateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /* Simplify drawing to guarantee the layer is the only thing drawn - so e.g. no background,
        scrolling, or fading edges. This guarantees all drawing is in the layer, so drawing
        properties (alpha, layer paint) affect all of the content of a TextureView. */

        if (canvas.isHardwareAccelerated()) {
            RecordingCanvas recordingCanvas = (RecordingCanvas) canvas;

            TextureLayer layer = getTextureLayer();
            if (layer != null) {
                applyUpdate();
                applyTransformMatrix();

                mLayer.setLayerPaint(mLayerPaint); // ensure layer paint is up to date
                recordingCanvas.drawTextureLayer(layer);
            }
        }
    }
```

TextureView#draw的触发时机这里就不展开了，接下来的调用栈如下：  
[TextureView#draw][tvDrawLink]  
&emsp;[TextureView#getTextureLayer][getlayerLink]  
首先，要通过硬件加速环境拿到一个TextureLayer对象：  
&emsp;&emsp;[HardwareRenderer#createTextureLayer][createLayerLink]  
&emsp;&emsp;&emsp;[HardwareRenderer#nCreateTextureLayer][nCreateLayerLink]  
&emsp;&emsp;&emsp;&emsp;[RenderProxy::createTextureLayer][proxyCreateLayerLink]（我们知道，proxy的任务会交给RenderThread在渲染线程中做）  
&emsp;&emsp;&emsp;&emsp;&emsp;[CanvasContext::createTextureLayer][ctxCreateLayerLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaOpenGLPipeline::createTextureLayer][pipeCreateLayerLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[DeferredLayerUpdater::DeferredLayerUpdater][deferredLayerUpdateLink]  
以上，TextureView通过ThreadedRender创建了一个DeferredLayerUpdater对象，并且把这个对象的地址作为以jlong的形式传递到TextureLayer中。至于DeferredLayerUpdater中有什么信息、DeferredLayerUpdater的职能是什么我们还需要向下看。  
&emsp;&emsp;&emsp;[TextureLayer#adoptTextureLayer][adoptTextLayerLink]  
&emsp;&emsp;&emsp;&emsp;[TextureLayer#TextureLayer][tLayerLink]  
TextureLayer主要保存的信息只有两个：HardwareRenderer对象TextureLayer#mRenderer和DeferredLayerUpdater对象的地址TextureLayer#mFinalizer。  
下面来看SurfaceTexture的创建：  
&emsp;&emsp;[SurfaceTexture#SurfaceTexture][SfTConstructorLink]  
&emsp;&emsp;&emsp;[SurfaceTexture#nativeInit][nativeInitLink]  
&emsp;&emsp;&emsp;&emsp;[SurfaceTexture::SurfaceTexture][nativeSFTLink]  
&emsp;&emsp;&emsp;&emsp;[SurfaceTexture_setSurfaceTexture][SurfaceTexture_setSurfaceTextureLink]给SurfaceTexture(java)#mSurfaceTexture赋值  
&emsp;&emsp;&emsp;&emsp;[SurfaceTexture_setProducer][SurfaceTexture_setProducerLink]给SurfaceTexture(java)#mProducer赋值  
在SurfaceTexture#nativeInit中会创建一个BufferQueue，BufferQueue中的Consumer会传递给SurfaceTexture(c++)。也就是说，SurfaceTexture或持有SurfaceTexture对象的对象就是这次创建出的BufferQueue的消费者。  
&emsp;&emsp;[TextureView#nCreateNativeWindow][nCreateNativeWindowLink]  
（frameworks/native/libs/nativedisplay/surfacetexture/surface_texture.cpp）  
&emsp;&emsp;&emsp;[Surface::Surface][nativeSurfaceConstructorLink]（将[SurfaceTexture#nativeInit][nativeInitLink]过程中创建的producer传递给Surface(c++)）  
总结：SurfaceTexture保存有完整的buffer queue，其中SurfaceTexture(java)持有producer，SurfaceTexture(c++)持有consumer。同时Android系统还利用producer创建了Surface(c++)对象，并由TextureView#mNativeWindow记录保存。  
&emsp;&emsp;[TextureLayer#setSurfaceTexture][setSurfaceTextureLink1]  
&emsp;&emsp;&emsp;[TextureLayer#nSetSurfaceTexture][nSetSurfaceTextureLink]  
&emsp;&emsp;&emsp;&emsp;[DeferredLayerUpdater::setSurfaceTexture][setSurfaceTextureLink3]  
现在，[HardwareRenderer#createTextureLayer][createLayerLink]中创建的DeferredLayerUpdater持有了[TextureView#getTextureLayer][getlayerLink]中创建的SurfaceTexture对象，而且DeferredLayerUpdater似乎只对SurfaceTexture对象作为consumer感兴趣。  
在[TextureView#getTextureLayer][getlayerLink]中还有一段代码值得注意：  

```java
            if (mListener != null && createNewSurface) {
                mListener.onSurfaceTextureAvailable(mSurface, getWidth(), getHeight());
            }
```

如果此时是首次执行[TextureView#draw][tvDrawLink]（代码中的体现就是TextureView#mSurface此前为null），这个时候就会调用回调，这是App开发中用的一个接口。  
&emsp;[RecordingCanvas#drawTextureLayer][rcDrawTextureLayerLink]  
&emsp;&emsp;[RecordingCanvas#nDrawTextureLayer][nDrawTextureLayerLink]（这里的第二个参数是通过TextureLayer#getLayerHandle获取的，实际上就是TextureLayer#mFinalizer，也就是DeferredLayerUpdater对象的唯一地址，这里不得不吐槽代码作者命名的曲折，让人没有办法一眼看出来）  
&emsp;&emsp;&emsp;[SkiaRecordingCanvas::drawLayer][nativeCanvasDrawLayerLink]（这里只是推测Canvas(c++)的实现是SkiaRecordingCanvas(c++)，但是并没有证据）  
这里有关“draw”的调用链条太过复杂，当然根本原因在于我还没有弄清楚Canvas这个概念所起的作用，因此我们不展开讨论，仅看和DeferredLayerUpdater有关的内容。  
[LayerDrawable::onDraw][LayerDrawableOnDrawLink]  
[LayerDrawable::DrawLayer][DrawLayerLink]  
涉及DeferredLayerUpdater的就是获取DeferredLayerUpdater中持有的[android::uirenderer::Layer][LayerLink]对象。但是，DeferredLayerUpdater::Layer是什么时候被赋值和修改的呢？上面我们只涉及到了DeferredLayerUpdater的实例化呀！！！

## 附

### 1. Canvas的继承关系

[RecordingCanvas][RecordingCanvasLink] -> [DisplayListCanvas][DisplayListCanvasLink] -> [BaseRecordingCanvas][BaseRecordingCanvasLink] -> [Canvas][CanvasLink] -> [BaseCanvas][BaseCanvasLink]  
[SkiaRecordingCanvas][SkiaRecordingCanvasLink] -> [SkiaCanvas][SkiaCanvasLink] -> [Canvas][NativeCanvasLink]

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

[tvDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureView.java;l=341
[getlayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureView.java;l=385
[createLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/HardwareRenderer.java;l=669
[nCreateLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp;l=265
[proxyCreateLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderProxy.cpp;l=151
[ctxCreateLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/CanvasContext.cpp;l=698
[pipeCreateLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp;l=142
[deferredLayerUpdateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/DeferredLayerUpdater.cpp;l=36
[adoptTextLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureLayer.java;l=144
[tLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureLayer.java;l=39
[SfTConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/SurfaceTexture.java;l=166
[nativeInitLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_graphics_SurfaceTexture.cpp;l=253
[nativeSFTLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/nativedisplay/surfacetexture/SurfaceTexture.cpp;l=61
[nCreateNativeWindowLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_view_TextureView.cpp;l=83
[SurfaceTexture_setSurfaceTextureLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_graphics_SurfaceTexture.cpp;l=84
[SurfaceTexture_setProducerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_graphics_SurfaceTexture.cpp;l=98
[nativeSurfaceConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/Surface.cpp;l=66
[setSurfaceTextureLink1]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureLayer.java;l=133
[nSetSurfaceTextureLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_TextureLayer.cpp;l=53
[setSurfaceTextureLink3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/DeferredLayerUpdater.cpp;l=53

[rcDrawTextureLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java;l=228
[nDrawTextureLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_DisplayListCanvas.cpp;l=144
[nativeCanvasDrawLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp;l=108

[RecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java
[DisplayListCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/DisplayListCanvas.java;l=31
[BaseRecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/BaseRecordingCanvas.java;l=41
[CanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/Canvas.java;l=52
[BaseCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/BaseCanvas.java;l=43

[SkiaRecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.h
[SkiaCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/SkiaCanvas.h
[NativeCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/hwui/Canvas.h

[LayerDrawableOnDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/LayerDrawable.cpp;l=29
[DrawLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/LayerDrawable.cpp;l=70

[LayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/Layer.h
