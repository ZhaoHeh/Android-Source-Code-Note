# Android的硬件加速渲染机制探究2

[ViewRootImpl#performTraversals][performTraversalsLink]  
&emsp;[ViewRootImpl#performDraw][performDrawLink]  
&emsp;&emsp;[ViewRootImpl#draw][drawLink]  

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

&emsp;&emsp;&emsp;[ThreadedRenderer#draw][ThreadedRendererDrawLink]  
&emsp;&emsp;&emsp;&emsp;[ThreadedRenderer#updateRootDisplayList][updateRootDisplayListLink]  

## 1. UPDATE

```java
    private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
        updateViewTreeDisplayList(view);

        // Consume and set the frame callback after we dispatch draw to the view above, but before
        // onPostDraw below which may reset the callback for the next frame.  This ensures that
        // updates to the frame callback during scroll handling will also apply in this frame.
        if (mNextRtFrameCallbacks != null) {
            final ArrayList<FrameDrawingCallback> frameCallbacks = mNextRtFrameCallbacks;
            mNextRtFrameCallbacks = null;
            setFrameCallback(frame -> {
                for (int i = 0; i < frameCallbacks.size(); ++i) {
                    frameCallbacks.get(i).onFrameDraw(frame);
                }
            });
        }

        if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
            RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
            try {
                final int saveCount = canvas.save();
                canvas.translate(mInsetLeft, mInsetTop);
                callbacks.onPreDraw(canvas);

                canvas.enableZ();
                canvas.drawRenderNode(view.updateDisplayListIfDirty());
                canvas.disableZ();

                callbacks.onPostDraw(canvas);
                canvas.restoreToCount(saveCount);
                mRootNodeNeedsUpdate = false;
            } finally {
                mRootNode.endRecording();
            }
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
// 代码路径：frameworks/base/core/java/android/view/ThreadedRenderer.java
```

1. display list在Java层的体现就是[android.graphics.RenderNode][RenderNodeLink]类；ThreadedRenderer持有window对应的RenderNode：[HardwareRenderer#mRootNode][mRootNodeLink]；DecorView持有root view对应的RenderNode：[View#mRenderNode][ViewMRenderNodeLink]。

2. **更新**当前window的display list，就是将root view的display list记录到window的display list中，实际的操作就是用canvas的draw方法修改native的数据。

3. **更新**的前提条件是```mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()```。

## 2. mRootNode.beginRecording

&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNode#beginRecording][beginRecordingLink]
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#obtain][RecordingCanvasObtainLink]
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#RecordingCanvas][rcConstructorLink]
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#nCreateDisplayListCanvas][nCreateDisplayListCanvasLink]
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Canvas::create_recording_canvas][create_recording_canvasLink]
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaRecordingCanvas::SkiaRecordingCanvas][SkiaRecordingCanvasLink]
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaRecordingCanvas::initDisplayList][initDisplayListLink]

## 附

### 中英文对照

window：窗口  
view：视图  

### 参考链接

1. 《Android应用程序UI硬件加速渲染的Display List构建过程分析》 by 罗升阳：<https://blog.csdn.net/luoshengyang/article/details/45943255>

2. 《Android应用程序UI硬件加速渲染的Display List渲染过程分析》 by 罗升阳：<https://blog.csdn.net/luoshengyang/article/details/46281499>

[performTraversalsLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=2350
[performDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=3776
[drawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewRootImpl.java;l=3969

[ThreadedRendererDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ThreadedRenderer.java;l=638
[updateRootDisplayListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ThreadedRenderer.java;l=563

[RenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RenderNode.java;l=194
[mRootNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/HardwareRenderer.java;l=148
[ViewMRenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/View.java;l=5183

[beginRecordingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RenderNode.java;l=370
[RecordingCanvasObtainLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java;l=57
[rcConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java;l=94
[nCreateDisplayListCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_DisplayListCanvas.cpp;l=106
[create_recording_canvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/hwui/Canvas.cpp;l=32
[SkiaRecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.h;l=33
[initDisplayListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp;l=42
