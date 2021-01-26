# TextureView

[TextureView#draw][TextureViewDrawLink]  

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
// 文件路径：frameworks/base/core/java/android/view/TextureView.java
```

### 6.1

&emsp;[TextureView#getTextureLayer][getTextureLayerLink]  

&emsp;&emsp;[HardwareRenderer#createTextureLayer][createLayerLink]  
&emsp;&emsp;&emsp;[HardwareRenderer#nCreateTextureLayer][nCreateLayerLink]  
&emsp;&emsp;&emsp;&emsp;[RenderProxy::createTextureLayer][proxyCreateLayerLink]  

（我们知道，proxy的任务会交给RenderThread在渲染线程中做，那么渲染线程是什么时机触发这次任务执行呢？主线程的performDraw是由VSync信号触发，那么渲染线程的任务就应该至少在下一个VSync信号到来时才能被执行吧？那又如何确保同步呢？毕竟还要返回一个对象的指针参数哇？）  

&emsp;&emsp;&emsp;&emsp;&emsp;[CanvasContext::createTextureLayer][ctxCreateLayerLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaOpenGLPipeline::createTextureLayer][pipeCreateLayerLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderThread::requireGlContext][requireGlContextLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderThread::renderState][GetRenderStateLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[DeferredLayerUpdater::DeferredLayerUpdater][deferredLayerUpdateLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderState::registerContextCallback][registerContextCallbackLink]  
&emsp;&emsp;&emsp;[TextureLayer#adoptTextureLayer][adoptTextLayerLink]  
&emsp;&emsp;&emsp;&emsp;[TextureLayer#TextureLayer][tLayerLink]  

[TextureViewDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureView.java;l=341
[getTextureLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureView.java;l=385
[createLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/HardwareRenderer.java;l=669
[nCreateLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp;l=265
[proxyCreateLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderProxy.cpp;l=151
[ctxCreateLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/CanvasContext.cpp;l=698
[pipeCreateLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp;l=142
[requireGlContextLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderThread.cpp;l=179
[GetRenderStateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderThread.h;l=104
[deferredLayerUpdateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/DeferredLayerUpdater.cpp;l=36
[registerContextCallbackLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderstate/RenderState.h;l=48
[adoptTextLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureLayer.java;l=144
[tLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureLayer.java;l=39

&emsp;&emsp;[SurfaceTexture#SurfaceTexture][SfTConstructorLink]  
&emsp;&emsp;&emsp;[SurfaceTexture#nativeInit][nativeInitLink]  
&emsp;&emsp;&emsp;&emsp;[SurfaceTexture::SurfaceTexture][nativeSFTLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[ConsumerBase::ConsumerBase][ConsumerBaseConstructorLink]  
&emsp;&emsp;&emsp;&emsp;[SurfaceTexture_setSurfaceTexture][ST_setSurfaceTextureLink]（这是直接地给Java层的SurfaceTexture#mSurfaceTexture赋值）  
&emsp;&emsp;&emsp;&emsp;[SurfaceTexture_setProducer][ST_setProducerLink]（这是直接地给Java层的SurfaceTexture#mProducer赋值）  
&emsp;&emsp;&emsp;&emsp;[SurfaceTexture_setFrameAvailableListener][ST_setFrameAvailableListenerLink]（这是直接地给Java层的SurfaceTexture#mFrameAvailableListener赋值）  

[SfTConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/SurfaceTexture.java;l=166
[nativeInitLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_graphics_SurfaceTexture.cpp;l=253
[nativeSFTLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/nativedisplay/surfacetexture/SurfaceTexture.cpp;l=61
[ConsumerBaseConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/ConsumerBase.cpp;l=58
[ST_setSurfaceTextureLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_graphics_SurfaceTexture.cpp;l=84
[ST_setProducerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_graphics_SurfaceTexture.cpp;l=98
[ST_setFrameAvailableListenerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_graphics_SurfaceTexture.cpp;l=112

&emsp;&emsp;[TextureView#nCreateNativeWindow][nCreateNativeWindowLink]  

[nCreateNativeWindowLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_view_TextureView.cpp;l=83

&emsp;&emsp;[TextureLayer#setSurfaceTexture][setSurfaceTextureLink1]  
&emsp;&emsp;&emsp;[TextureLayer#nSetSurfaceTexture][nSetSurfaceTextureLink]  
&emsp;&emsp;&emsp;&emsp;[DeferredLayerUpdater::setSurfaceTexture][setSurfaceTextureLink3]  

[setSurfaceTextureLink1]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureLayer.java;l=133
[nSetSurfaceTextureLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_TextureLayer.cpp;l=53
[setSurfaceTextureLink3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/DeferredLayerUpdater.cpp;l=53

&emsp;&emsp;[SurfaceTexture#setOnFrameAvailableListener][STsetOnFrameAvailableListenerLink]  
&emsp;&emsp;&emsp;[mAttachInfo][handlerTrace1].[mHandler][handlerTrace2]应该是主线程looper的一个handler  
&emsp;&emsp;&emsp;[TextureView#mUpdateListener][mUpdateListenerLink]在TextureView内部定义  

[STsetOnFrameAvailableListenerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/SurfaceTexture.java;l=201
[handlerTrace1]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=752
[handlerTrace2]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=5164
[mUpdateListenerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureView.java;l=806

### 6.2

&emsp;[TextureView#applyUpdate][applyUpdateLink]  
&emsp;&emsp;[TextureLayer#updateSurfaceTexture][updateSurfaceTextureLink]  
&emsp;&emsp;&emsp;[TextureLayer#nUpdateSurfaceTexture][nUpdateSurfaceTextureLink]  
&emsp;&emsp;&emsp;&emsp;[DeferredLayerUpdater::updateTexImage][updateTexImageLink]  
之所以要这样做，是因为纹理的更新要在Render Thread进行，而现在是在Main Thread执行。等到后面应用程序窗口的Display List被渲染时，TextureView的Open GL纹理才会被真正的更新。  
&emsp;&emsp;&emsp;[HardwareRenderer#pushLayerUpdate][pushLayerUpdateLink]  
&emsp;&emsp;&emsp;&emsp;[HardwareRenderer#nPushLayerUpdate][nPushLayerUpdateLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[RenderProxy::pushLayerUpdate][ProxyPushLayerUpdateLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[DrawFrameTask::pushLayerUpdate][DFTaskpushLayerUpdateLink]  
保存在这个列表中的DeferredLayerUpdater对象在渲染应用程序窗口的Display List的时候就会被处理.  

[applyUpdateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureView.java;l=458
[updateSurfaceTextureLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureLayer.java;drc=master;l=138
[nUpdateSurfaceTextureLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_TextureLayer.cpp;drc=master;l=60
[updateTexImageLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/DeferredLayerUpdater.h;drc=master;l=73
[pushLayerUpdateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/HardwareRenderer.java;drc=master;l=705
[nPushLayerUpdateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp;drc=master;l=288
[ProxyPushLayerUpdateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderProxy.cpp;drc=master;l=169
[DFTaskpushLayerUpdateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;drc=master;l=47

### 6.3

&emsp;[RecordingCanvas#drawTextureLayer][drawTextureLayerLink]  
&emsp;&emsp;[RecordingCanvas#nDrawTextureLayer][nDrawTextureLayerLink]  
&emsp;&emsp;&emsp;[SkiaRecordingCanvas::drawLayer][SkiaRecordingCanvasdrawLayerLink]  
&emsp;&emsp;&emsp;&emsp;[SkiaCanvas::drawDrawable][SkiaCanvasdrawDrawableLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[SkCanvas::drawDrawable][SkCanvasDrawDrawableLink]  

[drawTextureLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java;l=228;drc=master
[nDrawTextureLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_DisplayListCanvas.cpp;drc=master;l=144
[SkiaRecordingCanvasdrawLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_DisplayListCanvas.cpp;drc=master;l=144
[SkiaCanvasdrawDrawableLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/SkiaCanvas.h;drc=master;l=163
[SkCanvasDrawDrawableLink]:https://cs.android.com/android/platform/superproject/+/master:external/skia/src/core/SkCanvas.cpp;drc=master;l=2853

### 6.4 SurfaceTexture使用者如何更新

[SurfaceTexture#postEventFromNative][postEventFromNativeLink]  
&emsp;[mOnFrameAvailableHandler#handleMessage][handleMessageLink]  
&emsp;&emsp;[TextureView#mUpdateListener#onFrameAvailable][onFrameAvailableLink]  
&emsp;&emsp;&emsp;[TextureView#updateLayer][updateLayerLink]  

[postEventFromNativeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/SurfaceTexture.java;l=395
[handleMessageLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/SurfaceTexture.java;l=211
[onFrameAvailableLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureView.java;l=807
[updateLayerLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/TextureView.java;l=445

### 6.5 相机那头如何使用

[MainActivity#createSessionForPreviewFlow][createSessionForPreviewFlowLink]  
&emsp;[CameraDeviceImpl#createCaptureSession][createCaptureSessionLink]  
&emsp;&emsp;[CameraDeviceImpl#createCaptureSessionInternal][createCaptureSessionInternalLink]  

[createSessionForPreviewFlowLink]:https://github.com/ZhaoHeh/Android-Camera2-Test/blob/master/app/src/main/java/com/test/camera/MainActivity.java#L332
[createCaptureSessionLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/hardware/camera2/impl/CameraDeviceImpl.java;l=520
[createCaptureSessionInternalLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/hardware/camera2/impl/CameraDeviceImpl.java;drc=master;l=649