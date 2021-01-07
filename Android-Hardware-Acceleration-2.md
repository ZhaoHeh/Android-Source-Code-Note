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

## 1. ThreadedRenderer#updateRootDisplayList

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

## 2. mRootNode.beginRecording：called by ThreadedRenderer#updateRootDisplayList

&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNode#beginRecording][beginRecordingLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#obtain][RecordingCanvasObtainLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#RecordingCanvas][rcConstructorLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#nCreateDisplayListCanvas][nCreateDisplayListCanvasLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Canvas::create_recording_canvas][create_recording_canvasLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaRecordingCanvas::SkiaRecordingCanvas][SkiaRecordingCanvasLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaRecordingCanvas::initDisplayList][initDisplayListLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNode::detachAvailableList][detachAvailableListLink]

（1）RenderNode#beginRecording会得到一个RecordingCanvas对象，RecordingCanvas对象在native层对应一个SkiaRecordingCanvas对象，SkiaRecordingCanvas对象持有从window的RenderNode(c++)对象中获取的display list，SkiaRecordingCanvas::mDisplayList = RenderNode::mAvailableDisplayList  
（2）RenderNode::detachAvailableList中的std::move的含义是什么？  
（3）RenderNode#beginRecording到底做了什么？我似乎还没有把握到核心点  

## 3. canvas.drawRenderNode：called by ThreadedRenderer#updateRootDisplayList

&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#drawRenderNode][RCdrawRenderNodeLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#nDrawRenderNode][nDrawRenderNodeLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaRecordingCanvas::drawRenderNode][nativeSRCdrawRenderNodeLink]  

```c++
void SkiaRecordingCanvas::drawRenderNode(uirenderer::RenderNode* renderNode) {
    // Record the child node. Drawable dtor will be invoked when mChildNodes deque is cleared.
    mDisplayList->mChildNodes.emplace_back(renderNode, asSkCanvas(), true, mCurrentBarrier);
    auto& renderNodeDrawable = mDisplayList->mChildNodes.back();
    if (Properties::getRenderPipelineType() == RenderPipelineType::SkiaVulkan) {
        // Put Vulkan WebViews with non-rectangular clips in a HW layer
        renderNode->mutateStagingProperties().setClipMayBeComplex(mRecorder.isClipMayBeComplex());
    }
    drawDrawable(&renderNodeDrawable);

    // use staging property, since recording on UI thread
    if (renderNode->stagingProperties().isProjectionReceiver()) {
        mDisplayList->mProjectionReceiver = &renderNodeDrawable;
    }
}
// 文件路径：frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp
```

（1）这里的入参```uirenderer::RenderNode*```是root view的RenderNode(java)对应的RenderNode(c++)对象  
（2）[SkiaDisplayList::mChildNodes][mChildNodesLink]类型为```std::deque<RenderNodeDrawable>```，是一个RenderNodeDrawable队列，std::deque<T,Allocator>::emplace_back的含义是“Appends a new element to the end of the container.”，也就是说上述操作先利用```renderNode, asSkCanvas(), true, mCurrentBarrier```四个参数new了一个RenderNodeDrawable对象，然后将RenderNodeDrawable对象添加到SkiaDisplayList::mChildNodes队列尾部。  
（3）std::deque<T,Allocator>::back的含义是“Returns a reference to the last element in the container.”，也就是拿到了刚刚new的RenderNodeDrawable对象。  

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNodeDrawable::RenderNodeDrawable][RenderNodeDrawableConstructorLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaCanvas::drawDrawable][SkiaCanvasDrawDrawableLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkCanvas::drawDrawable][SkCanvasDrawDrawableLink]（注意SkCanvas::drawDrawable的[函数声明][FunctionDeclarationLink]，第二个参数可以不显示指定）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkCanvas::onDrawDrawable][SkCanvasOnDrawDrawableLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkBaseDevice::drawDrawable][SkBaseDeviceDrawDrawableLink]（这一层调用的意义是什么？？？） 

下面高能，这几个函数没一个看得懂的：  

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkDrawable::draw][SkDrawableDrawLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNodeDrawable::onDraw][RenderNodeDrawableOnDrawLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNodeDrawable::forceDraw][RenderNodeDrawableForceDrawLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNodeDrawable::drawContent][RenderNodeDrawableDrawContentLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaDisplayList::draw][SkiaDisplayListDrawLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[DisplayListData::draw][DisplayListDataDrawLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[DisplayListData::map][DisplayListDataDrawLink]  

上面的函数虽然一个也看不懂，但是我们可以通过分析类之间的关系和参数的传递在确定整个RecordingCanvas#drawRenderNode过程到底是在做什么：  
（1）当前window会拥有一个root render node，在开始draw之前，会通过RenderNode#beginRecording获得一个和window对应的canvas，这个canvas包含的内容很广，包括Java层的canvas，Java层对应的native层的canvas，native层canvas持有的Skia模块的sk canvas。我们可以把Java层的canvas和native层的canvas看成一个整体（比如称为android app的window的canvas），但是不能这么看待sk canvas，可以说sk canvas是android app的window的canvas持有的一个有关Skia模块的工具。  
（2）android app的window的canvas会在native层持有一个display list，这是其对应的render node传递给他的，与其说canvas持有一个对应的display list，不如说在draw时，render node会将自己对应的display list交给canvas暂时保管与使用。  
（3）root view也持有一个对应的render node，render node也会有一个对应的display list。  
（4）RecordingCanvas#drawRenderNode就是一个把root view对应的display list map到window此时对应的那个canvas中持有的sk canvas中的过程。  
（5）这里面细节很多，我们暂时顾不上，只能以后再说，但是注意，Skia和OpenGL的联系我们还没找到呢！！！

## 附

### Canvas的继承关系

[RecordingCanvas][RecordingCanvasLink] -> [DisplayListCanvas][DisplayListCanvasLink] -> [BaseRecordingCanvas][BaseRecordingCanvasLink] -> [Canvas][CanvasLink] -> [BaseCanvas][BaseCanvasLink]  
[SkiaRecordingCanvas][SkiaRecordingCanvasLink] -> [SkiaCanvas][SkiaCanvasLink] -> [Canvas][NativeCanvasLink]

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
[detachAvailableListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.h;l=290

[mChildNodesLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaDisplayList.h;l=154

[RCdrawRenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java;l=214
[nDrawRenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_DisplayListCanvas.cpp;l=138
[nativeSRCdrawRenderNodeLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp;l=119
[RenderNodeDrawableConstructorLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/RenderNodeDrawable.cpp;l=29
[SkiaCanvasDrawDrawableLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/SkiaCanvas.h;l=163
[SkCanvasDrawDrawableLink]:https://cs.android.com/android/platform/superproject/+/master:external/skia/src/core/SkCanvas.cpp;l=2853
[FunctionDeclarationLink]:https://cs.android.com/android/platform/superproject/+/master:external/skia/include/core/SkCanvas.h;l=2443
[SkCanvasOnDrawDrawableLink]:https://cs.android.com/android/platform/superproject/+/master:external/skia/src/core/SkCanvas.cpp;l=2864
[SkBaseDeviceDrawDrawableLink]:https://cs.android.com/android/platform/superproject/+/master:external/skia/src/core/SkDevice.cpp;l=304
[SkDrawableDrawLink]:https://cs.android.com/android/platform/superproject/+/master:external/skia/src/core/SkDrawable.cpp;l=33
[RenderNodeDrawableOnDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/RenderNodeDrawable.cpp;l=109
[RenderNodeDrawableForceDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/RenderNodeDrawable.cpp;l=135
[RenderNodeDrawableDrawContentLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/RenderNodeDrawable.cpp;l=203
[SkiaDisplayListDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaDisplayList.h;l=143
[DisplayListDataDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RecordingCanvas.cpp;l=740

[RecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java
[DisplayListCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/DisplayListCanvas.java;l=31
[BaseRecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/BaseRecordingCanvas.java;l=41
[CanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/Canvas.java;l=52
[BaseCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/BaseCanvas.java;l=43

[SkiaRecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.h
[SkiaCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/SkiaCanvas.h
[NativeCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/hwui/Canvas.h
