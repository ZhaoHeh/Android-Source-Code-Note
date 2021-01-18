# Android的硬件加速渲染机制探究3

## 1 引子

[ViewRootImpl#performTraversals][performTraversalsLink]  
&emsp;[ViewRootImpl#performDraw][performDrawLink]  
&emsp;&emsp;[ViewRootImpl#draw][drawLink]，代码如下：  

[performTraversalsLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=2350
[performDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=3776
[drawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=3969

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
// 代码路径：frameworks/base/core/java/android/view/ViewRootImpl.java
```

&emsp;&emsp;&emsp;[ThreadedRenderer#draw][ThreadedRendererDrawLink]，代码如下：  

[ThreadedRendererDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ThreadedRenderer.java;l=638

```java
    void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
        final Choreographer choreographer = attachInfo.mViewRootImpl.mChoreographer;
        choreographer.mFrameInfo.markDrawStart();

        updateRootDisplayList(view, callbacks);

        // register animating rendernodes which started animating prior to renderer
        // creation, which is typical for animators started prior to first draw
        if (attachInfo.mPendingAnimatingRenderNodes != null) {
            final int count = attachInfo.mPendingAnimatingRenderNodes.size();
            for (int i = 0; i < count; i++) {
                registerAnimatingRenderNode(
                        attachInfo.mPendingAnimatingRenderNodes.get(i));
            }
            attachInfo.mPendingAnimatingRenderNodes.clear();
            // We don't need this anymore as subsequent calls to
            // ViewRootImpl#attachRenderNodeAnimator will go directly to us.
            attachInfo.mPendingAnimatingRenderNodes = null;
        }

        int syncResult = syncAndDrawFrame(choreographer.mFrameInfo);
        if ((syncResult & SYNC_LOST_SURFACE_REWARD_IF_FOUND) != 0) {
            Log.w("OpenGLRenderer", "Surface lost, forcing relayout");
            // We lost our surface. For a relayout next frame which should give us a new
            // surface from WindowManager, which hopefully will work.
            attachInfo.mViewRootImpl.mForceNextWindowRelayout = true;
            attachInfo.mViewRootImpl.requestLayout();
        }
        if ((syncResult & SYNC_REDRAW_REQUESTED) != 0) {
            attachInfo.mViewRootImpl.invalidate();
        }
    }
// 代码路径：frameworks/base/core/java/android/view/ThreadedRenderer.java
```

## 2 ThreadedRenderer#syncAndDrawFrame的调用栈

&emsp;[ThreadedRenderer#syncAndDrawFrame][syncAndDrawFrameLink]  
&emsp;&emsp;[ThreadedRenderer#nSyncAndDrawFrame][nSyncAndDrawFrameLink]  
&emsp;&emsp;&emsp;[RenderProxy::syncAndDrawFrame][proxySyncAndDrawFrameLink]  
&emsp;&emsp;&emsp;&emsp;[DrawFrameTask::drawFrame][DrawFrameTaskDrawFrameLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[DrawFrameTask::postAndWait][DrawFrameTaskPostAndWaitLink]，实现如下：  

[syncAndDrawFrameLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/HardwareRenderer.java;drc=master;l=432
[nSyncAndDrawFrameLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp;drc=master;l=227
[proxySyncAndDrawFrameLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/RenderProxy.cpp;drc=master;l=120
[DrawFrameTaskDrawFrameLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;drc=master;l=68
[DrawFrameTaskPostAndWaitLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;drc=master;l=78

```c++
void DrawFrameTask::postAndWait() {
    AutoMutex _lock(mLock);
    mRenderThread->queue().post([this]() { run(); });
    mSignal.wait(mLock);
}
// 代码路径：frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp
```

## 3 render thread线程

DrawFrameTask::postAndWait就将当前正在处理的DrawFrameTask对象添加到render thread的task queue中，并且在另外一个成员变量mSignal描述的一个条件变量上进行等待。  
下面进入硬件渲染线程：  

[DrawFrameTask::run][DrawFrameTaskRunLink]  

[DrawFrameTaskRunLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;drc=master;l=84

```c++
void DrawFrameTask::run() {
    ATRACE_NAME("DrawFrame");

    bool canUnblockUiThread;
    bool canDrawThisFrame;
    {
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        canUnblockUiThread = syncFrameState(info);
        canDrawThisFrame = info.out.canDrawThisFrame;

        if (mFrameCompleteCallback) {
            mContext->addFrameCompleteListener(std::move(mFrameCompleteCallback));
            mFrameCompleteCallback = nullptr;
        }
    }

    // Grab a copy of everything we need
    CanvasContext* context = mContext;
    std::function<void(int64_t)> callback = std::move(mFrameCallback);
    mFrameCallback = nullptr;

    // From this point on anything in "this" is *UNSAFE TO ACCESS*
    if (canUnblockUiThread) {
        unblockUiThread();
    }

    // Even if we aren't drawing this vsync pulse the next frame number will still be accurate
    if (CC_UNLIKELY(callback)) {
        context->enqueueFrameWork(
                [callback, frameNr = context->getFrameNumber()]() { callback(frameNr); });
    }

    if (CC_LIKELY(canDrawThisFrame)) {
        context->draw();
    } else {
        // wait on fences so tasks don't overlap next frame
        context->waitOnFences();
    }

    if (!canUnblockUiThread) {
        unblockUiThread();
    }
}

void DrawFrameTask::unblockUiThread() {
    AutoMutex _lock(mLock);
    mSignal.signal();
}

// 代码路径：frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp
```

说明：  
（1）从代码中可知，main thread请求render thread执行DrawFrameTask的时候，不能马上返回，而是进入等待状态；等到render thread调用[DrawFrameTask::unblockUiThread][unblockUiThreadLink]之后，main thread才会被唤醒；  
（2）类[android::uirenderer::RenderNode][nativeRenderNodeLink]中，成员变量[mStagingProperties][mStagingPropertiesLink]和[mStagingDisplayList][mStagingDisplayListLink]由main thread维护，而成员变量[mProperties][mPropertiesLink]和[mDisplayList][mDisplayListLink]由render thread维护；  
（3）[当main thread构建完window的display list之后][nodeEndRecordingLink]，就会调用[RenderNode::setStagingDisplayList][nativeNodeSetStagingLink]将其设置到root render node的成员变量[RenderNode::mStagingDisplayList][mStagingDisplayListLink]中去；  
（4）而当应用程序窗口某一个View的Property发生变化时，就会调用RenderNode类的成员函数mutateStagingProperties获得成员变量mStagingProperties描述的Render Properties，进而修改相应的Property【未找到依据】；  
（5）当main thread维护的Render Properties发生变化时，成员变量mDirtyPropertyFields的值就不等于0，其中不等于0的位就表示是哪一个具体的Property发生了变化，而当main thread维护的Display List Data发生变化时，成员变量mNeedsDisplayListDataSync的值就等于true，表示要从main thread同步到Render Thread【未找到依据】。  

[unblockUiThreadLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;l=166;drc=master
[nativeRenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.h;l=76
[nodeEndRecordingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RenderNode.java;l=410
[nativeNodeSetStagingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.cpp;l=77
[mStagingPropertiesLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.h;l=247
[mStagingDisplayListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.h;l=256
[mPropertiesLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.h;l=246
[mDisplayListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.h;l=255

### 3.1 DrawFrameTask::syncFrameState

&emsp;[DrawFrameTask::syncFrameState][taskSyncFrameStateLink]  

&emsp;&emsp;[CanvasContext::makeCurrent][CanvasContextMakeCurrentLink]  
&emsp;&emsp;&emsp;&emsp;[SkiaOpenGLPipeline::makeCurrent][SkiaOpenGLPipelineMakeCurrentLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[EglManager::makeCurrent][EglManagerMakeCurrentLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[eglMakeCurrent][eglMakeCurrentLink]  

makeCurrent的机器意义是什么呢？  

[taskSyncFrameStateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp;drc=master;l=128
[CanvasContextMakeCurrentLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/CanvasContext.cpp;drc=master;l=250
[SkiaOpenGLPipelineMakeCurrentLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp;l=56
[EglManagerMakeCurrentLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/renderthread/EglManager.cpp;drc=master;l=401
[eglMakeCurrentLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/eglApi.cpp;l=183

&emsp;&emsp;[DeferredLayerUpdater::apply][DeferredLayerUpdaterApplyLink]  

[DeferredLayerUpdaterApplyLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/DeferredLayerUpdater.cpp;l=121

### 3.2 CanvasContext::draw
