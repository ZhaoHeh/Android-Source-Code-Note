# Android的硬件加速渲染机制探究

## 1. 初始化

```java
    /**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
        synchronized (this) {
            if (mView == null) {
                // ... 省略代码
                // If the application owns the surface, don't enable hardware acceleration
                if (mSurfaceHolder == null) {
                    // While this is supposed to enable only, it can effectively disable
                    // the acceleration too.
                    enableHardwareAcceleration(attrs);
                    final boolean useMTRenderer = MT_RENDERER_AVAILABLE
                            && mAttachInfo.mThreadedRenderer != null;
                    if (mUseMTRenderer != useMTRenderer) {
                        // Shouldn't be resizing, as it's done only in window setup,
                        // but end just in case.
                        endDragResizing();
                        mUseMTRenderer = useMTRenderer;
                    }
                }
                // ... 省略代码
            }
        }
    }
// 代码路径：frameworks/base/core/java/android/view/ViewRootImpl.java
```

调用时机：SubActivity#onCreate -> SubActivity#setContentView -> ... -> [ViewRootImpl#setView][setView] -> [ViewRootImpl#enableHardwareAcceleration][enableHardwareAcceleration]  

从ViewRootImpl#enableHardwareAcceleration开始的调用栈：  

[ViewRootImpl#enableHardwareAcceleration][enableHardwareAcceleration]  
&emsp;[ThreadedRenderer#create][ThreadedRendererCreadte]  
&emsp;&emsp;[ThreadedRenderer#ThreadedRenderer][ThreadedRendererConstructorLink]  
&emsp;&emsp;&emsp;[HardwareRenderer#HardwareRenderer][HardwareRenderer]  
下面会进入native层执行，实例化一系列C++对象：  
&emsp;&emsp;&emsp;&emsp;[HardwareRenderer#nCreateRootRenderNode][nCreateRootRenderNodeLink]（创建窗口的render node也即root render node）  
&emsp;&emsp;&emsp;&emsp;&emsp;[RootRenderNode::RootRenderNode][RootRenderNodeLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNode::RenderNode][RenderNodeLink]  
&emsp;&emsp;&emsp;&emsp;[HardwareRenderer#nCreateProxy][nCreateProxyLink]（代理render proxy的创建过程）  
&emsp;&emsp;&emsp;&emsp;&emsp;[RenderProxy::RenderProxy][RenderProxyLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderThread::getInstance][RenderThreadGetInstanceLink]（给RenderProxy::mRenderThread赋值）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderThread::RenderThread][RenderThreadConstructorLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[CanvasContext::create][CanvasContextCreateLink]（给RenderProxy::mContext赋值）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[CanvasContext::CanvasContext][CanvasContextLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[DrawFrameTask::setContext][DrawFrameTaskSetCtxLink]（？？？mDrawFrameTask是什么时候被赋值的呢）  

[setView]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=977
[enableHardwareAcceleration]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=1298
[ThreadedRendererCreadte]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ThreadedRenderer.java;l=252
[ThreadedRendererConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ThreadedRenderer.java;l=288
[HardwareRenderer]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/HardwareRenderer.java;l=157
[nCreateRootRenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp;l=138
[RootRenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RootRenderNode.h;l=32
[RenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.cpp;l=61
[nCreateProxyLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp;l=145
[RenderProxyLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderProxy.cpp;l=36
[RenderThreadGetInstanceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderThread.cpp;l=118
[RenderThreadConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderThread.cpp;l=127
[CanvasContextCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/CanvasContext.cpp;l=59
[CanvasContextLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/CanvasContext.cpp;l=97
[DrawFrameTaskSetCtxLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;l=40

下面进行一些名词概念的约定，以方便说明上述代码的具体流程：  

main thread : App的UI主线程  
root view impl : android.view.ViewRootImpl;  
threaded renderer : android.view.ThreadedRenderer; android.graphics.HardwareRenderer;  
root render node : HardwareRenderer#mRootNode; RenderNode#mNativeRenderNode; RootRenderNode(cpp); RenderNode(cpp);  
render proxy : HardwareRenderer#mNativeProxy; android::uirenderer::renderthread::RenderProxy;  
render thread : android::uirenderer::renderthread::RenderThread; android::uirenderer::ThreadBase; android::Thread;  

ViewRootImpl#setView是被Activity#OnCreate调用到的，Activity#OnCreate是在main thread中进行的，因此以下逻辑是从main thread开始的：  
root view impl实例化一个threaded renderer，threaded renderer持有两个对象，分别是root render node和render proxy。  
render proxy

Android应用程序UI硬件加速渲染环境的核心可以说是Render Thread，下面的内容就围绕Render Thread展开论述，主要的关注点就是：  
（1）初始化：创建Render Thread的过程  
（2）初始化2：通过Render Thread创建EGL Surface的过程  
（3）业务执行：通过Render Thread渲染一帧画面的过程  

这两个函数会牵扯出几个重要的native类:  
[RenderNode][RenderNodeLink](Path: frameworks/base/libs/hwui/RenderNode.cpp)  
[RenderProxy][RenderProxyLink](Path: frameworks/base/libs/hwui/renderthread/RenderProxy.cpp)  
[RenderThread][RenderThreadLink](Path: frameworks/base/libs/hwui/renderthread/RenderThread.cpp)  

关于[RenderProxy][RenderProxyLink]需要了解如下几个要点：  
（1）RenderThread的初始化在RenderProxy的构造函数中完成  
（2）RenderProxy向RenderThread发送的第一个命令就是让其创建[CanvasContext][CanvasContextLink]对象  
（3）RenderProxy主要包含三个重要的成员变量：mRenderThread mContext mDrawFrameTask  
（4）[CanvasContext的创建][CanvasContextCreateLink]在RenderThread线程中完成，CanvasContext中最重要的成员就是mRenderPipeline  

关于[RenderThread][RenderThreadLink]需要重点关注如下几点：  
（1）RenderThread依赖Android的native Looper机制实现  
（2）RenderThread在[RenderThread::threadLoop()][threadLoopLink]中调用[RenderThread::initThreadLocals()][initThreadLocalsLink]  
（3）RenderThread中两个成员变量mEglManager和mVkManager涉及GPU渲染的API（OpenGL和Vulkan）  
（4）RenderThread依赖vsync信号  
（5）RenderThread具体的事件驱动机制目前并不清楚，老罗介绍的代码是比较旧的了  

## 2. 绑定窗口到Render Thread

```java
    private void performTraversals() {
        // ... 省略代码
        if (mFirst || windowShouldResize || viewVisibilityChanged || cutoutChanged || params != null
                || mForceNextWindowRelayout) {
            // ... 省略代码
            try {
                // ... 省略代码
                relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
                // ... 省略代码
                if (surfaceCreated) {
                    // If we are creating a new surface, then we need to
                    // completely redraw it.
                    mFullRedrawNeeded = true;
                    mPreviousTransparentRegion.setEmpty();

                    // Only initialize up-front if transparent regions are not
                    // requested, otherwise defer to see if the entire window
                    // will be transparent
                    if (mAttachInfo.mThreadedRenderer != null) {
                        try {
                            hwInitialized = mAttachInfo.mThreadedRenderer.initialize(mSurface);
                            if (hwInitialized && (host.mPrivateFlags
                                            & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) == 0) {
                                // Don't pre-allocate if transparent regions
                                // are requested as they may not be needed
                                mAttachInfo.mThreadedRenderer.allocateBuffers();
                            }
                        } catch (OutOfResourcesException e) {
                            handleOutOfResourcesException(e);
                            return;
                        }
                    }
                    notifySurfaceCreated();
                } else if (surfaceDestroyed) {
        // ... 省略代码
        mIsInTraversal = false;
    }
```

函数调用栈：  
[ViewRootImpl#performTraversals][performTraversalsLink]  
&emsp;[ThreadedRenderer#initialize][RendererinitializeLink]  
&emsp;&emsp;[ThreadedRenderer#setSurface][RendererSetSurfaceLink]  
&emsp;&emsp;&emsp;[HardwareRenderer#setSurface][RendererSetSurfaceLink1]  
&emsp;&emsp;&emsp;&emsp;[HardwareRenderer#nSetSurface][nSetSurfaceLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[RenderProxy::setSurface][ProxySetSurfaceLink]  
下面开始进入RenderThread执行：  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[CanvasContext::setSurface][CtxSetSurfaceLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaOpenGLPipeline::setSurface][PipelineSetSurfaceLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderThread::requireGlContext][requireGlContextLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[EglManager::initialize][EglMgrInitLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[eglGetDisplay][eglGetDisplayLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[eglInitialize][eglInitializeLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[EglManager::createContext][EglMgrCreateCtxLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[eglCreateContext][eglCreateContextLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[EglManager::createPBufferSurface][EglMgrCreatePBSLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[eglCreatePbufferSurface][eglCreatePbufferSurfaceLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[EglManager::makeCurrent][EglMgrMkCurrentLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[eglMakeCurrent][eglMakeCurrentLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[eglSwapInterval][eglSwapIntervalLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[EglManager::createSurface][EglMgrCreateSfcLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[eglCreateWindowSurface][eglCreateWindowSurfaceLink]  

[performTraversalsLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=2718
[RendererinitializeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ThreadedRenderer.java;l=361
[RendererSetSurfaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ThreadedRenderer.java;l=410
[RendererSetSurfaceLink1]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/HardwareRenderer.java;l=299
[nSetSurfaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp;l=174
[ProxySetSurfaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderProxy.cpp;l=79
[CtxSetSurfaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/CanvasContext.cpp;l=157
[PipelineSetSurfaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp;l=160
[requireGlContextLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderThread.cpp;l=179

[EglMgrInitLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/EglManager.cpp;l=101
[EglMgrCreateSfcLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/EglManager.cpp;l=309
[EglMgrCreateCtxLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/EglManager.cpp;l=278
[EglMgrCreatePBSLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/EglManager.cpp;l=295
[EglMgrMkCurrentLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/EglManager.cpp;l=401

[eglGetDisplayLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/eglApi.cpp;l=41
[eglInitializeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/eglApi.cpp;l=70
[eglCreateContextLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/eglApi.cpp;l=168
[eglCreatePbufferSurfaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/eglApi.cpp;l=139
[eglMakeCurrentLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/eglApi.cpp;l=183
[eglSwapIntervalLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/eglApi.cpp;l=311
[eglCreateWindowSurfaceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/eglApi.cpp;l=107

把老罗的原话整理一下，非常有用：  

1. Activity窗口的绘制流程是在ViewRootImpl#performTraversals发起的  
2. 在绘制之前，首先要通过ViewRootImpl#relayoutWindow向WindowManagerService申请一个surface  
3. 一旦获得了对应的Surface，就需要将它绑定到Render Thread中  

## 3. 渲染一帧画面

```java
    private boolean draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;
        // ... 省略代码
        mAttachInfo.mTreeObserver.dispatchOnDraw();
        // ... 省略代码
        mAttachInfo.mDrawingTime =
                mChoreographer.getFrameTimeNanos() / TimeUtils.NANOS_PER_MS;

        boolean useAsyncReport = false;
        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
                // ... 省略代码
                mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
            } else {
                // ... 省略代码
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
                }
            }
        }
        // ... 省略代码
        return useAsyncReport;
    }
```

上述代码之前的调用：  
[ViewRootImpl#performTraversals][performTraversalsLink3]  
&emsp;[ViewRootImpl#performDraw][performDrawLink3]  
&emsp;&emsp;[ViewRootImpl#draw][drawLink3]  

从上述代码开始继续调用：  
[ThreadedRenderer#draw][threadedRenderDrawLink3]  
&emsp;[HardwareRenderer#syncAndDrawFrame][syncAndDrawFrame3]  
&emsp;&emsp;[HardwareRenderer#nSyncAndDrawFrame][nSyncAndDrawFrame3]  
&emsp;&emsp;&emsp;[RenderProxy::syncAndDrawFrame][RenderProxySyncAndDrawFrame3]  
&emsp;&emsp;&emsp;&emsp;[DrawFrameTask::drawFrame][TaskDrawFrame3]  
&emsp;&emsp;&emsp;&emsp;&emsp;[DrawFrameTask::postAndWait][TaskPostAndWait3]  

下面开始进入RenderThread执行：  
[DrawFrameTask::run][DrawFrameTaskRun3]  
&emsp;[DrawFrameTask::syncFrameState][syncFrameState3]  
&emsp;&emsp;[CanvasContext::makeCurrent][ContextMakeCurrent3]  
&emsp;&emsp;&emsp;[SkiaOpenGLPipeline::makeCurrent][PipeMakeCurrent3]  
&emsp;&emsp;&emsp;&emsp;[EglManager::makeCurrent][EglMakeCurrent3]  

[RenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.cpp
[RenderProxyLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderProxy.cpp;l=36
[RenderThreadLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderThread.cpp;l=127
[threadLoopLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderThread.cpp;l=323
[initThreadLocalsLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderThread.cpp;l=164

[CanvasContextLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/CanvasContext.cpp;l=59
[CanvasContextCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/CanvasContext.cpp;l=59

[performTraversalsLink3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=3104
[performDrawLink3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=3833
[drawLink3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=4106
[threadedRenderDrawLink3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ThreadedRenderer.java;l=638
[syncAndDrawFrame3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/HardwareRenderer.java;l=432
[nSyncAndDrawFrame3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp;l=227
[RenderProxySyncAndDrawFrame3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderProxy.cpp;l=120
[TaskDrawFrame3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;l=68
[TaskPostAndWait3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;l=78
[DrawFrameTaskRun3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;l=84
[syncFrameState3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;l=128
[ContextMakeCurrent3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/CanvasContext.cpp;l=250
[PipeMakeCurrent3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp;l=56
[EglMakeCurrent3]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/EglManager.cpp;l=401

参考链接：  
[Android应用程序UI硬件加速渲染环境初始化过程分析](https://blog.csdn.net/luoshengyang/article/details/45769759)  
[EGLSurface 和 OpenGL ES](https://source.android.com/devices/graphics/arch-egl-opengl)
