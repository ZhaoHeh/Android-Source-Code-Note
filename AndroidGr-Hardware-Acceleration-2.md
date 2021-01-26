# Android的硬件加速渲染机制探究2

[ViewRootImpl#performTraversals][performTraversalsLink]  
&emsp;[ViewRootImpl#performDraw][performDrawLink]  
&emsp;&emsp;[ViewRootImpl#draw][drawLink]，代码如下：  

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

&emsp;&emsp;&emsp;&emsp;[ThreadedRenderer#updateRootDisplayList][updateRootDisplayListLink]，代码如下：  

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

先以几条未经代码证明的断言作为说明，以总括整篇文章，后面的分析都不会跳出出这几条断言的圈：  

1. display list在Java层的体现就是[android.graphics.RenderNode][RenderNodeLink]类；ThreadedRenderer持有window对应的RenderNode：[HardwareRenderer#mRootNode][mRootNodeLink]；DecorView持有root view对应的RenderNode：[View#mRenderNode][ViewMRenderNodeLink]。

2. **更新**当前window的display list，就是将root view的display list记录到window的display list中，实际的操作就是用canvas的draw方法修改native的数据。

3. canvas和display list的关系可以说是临时性的，canvas对象只有在draw的时候才会new或者向对象池申请；render node和display list的关系则是强绑定且一一对应的。

## **1. updateViewTreeDisplayList(view)：called by ThreadedRenderer#updateRootDisplayList**

&emsp;&emsp;&emsp;&emsp;&emsp;[ThreadedRenderer#updateViewTreeDisplayList][updateViewTreeDisplayListLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[View#updateDisplayListIfDirty][updateDisplayListIfDirtyLink]（注意：此时此刻的具体实现者是root view：DecorView）  

```java
    public RenderNode updateDisplayListIfDirty() {
        final RenderNode renderNode = mRenderNode;
        // ... 省略代码
        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
                || !renderNode.hasDisplayList()
                || (mRecreateDisplayList)) {
            // Don't need to recreate the display list, just need to tell our
            // children to restore/recreate theirs
            if (renderNode.hasDisplayList()
                    && !mRecreateDisplayList) {
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                dispatchGetDisplayList();

                return renderNode; // no work needed
            }

            // If we got here, we're recreating it. Mark it as such to ensure that
            // we copy in child display lists into ours in drawChild()
            mRecreateDisplayList = true;
            // ... 省略代码
            final RecordingCanvas canvas = renderNode.beginRecording(width, height);

            try {
                if (layerType == LAYER_TYPE_SOFTWARE) {
                    buildDrawingCache(true);
                    Bitmap cache = getDrawingCache(true);
                    if (cache != null) {
                        canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                    }
                } else {
                    // ... 省略代码
                        draw(canvas);
                }
            } finally {
                renderNode.endRecording();
                setDisplayListProperties(renderNode);
            }
        }
        // ... 省略代码
        return renderNode;
    }
// 文件路径：frameworks/base/core/java/android/view/View.java
```

```(mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0```表明上次构建的display list已经失效，需要重新构建：  

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[ViewGroup#dispatchGetDisplayList][ViewGroupDispatchGetDisplayListLink]（DecorView继承自ViewGroup）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[ViewGroup#recreateChildDisplayList][recreateChildDisplayListLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[View#updateDisplayListIfDirty][updateDisplayListIfDirtyLink]（注意：这里的具体实现应该是root view(DecorView)的一个个child们）  

总结：上述调用表明：此时root view什么也需要不做，只需要开始递归执行其child view们的updateDisplayListIfDirty函数。  

```!renderNode.hasDisplayList()```表明root view的render node内的display list data还没有设置过或者已经被销毁，```(mRecreateDisplayList)```则是一个直接出发重新构建display list的flag值，这两个条件任意一个为true都会触发如下调用栈：  

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNode#beginRecording][beginRecordingLink]（注意：这里的render node是root view的，所以获取的canvas也是root view的）  
如果```if (layerType == LAYER_TYPE_SOFTWARE)```（这个条件意味着什么？？？？），则：  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[BaseRecordingCanvas#drawBitmap][BRCanvasDrawBitmapLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[BaseRecordingCanvas#nDrawBitmap][android_graphics_Canvas_drawBitmapLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaRecordingCanvas::drawBitmap][SkiaRecordingCanvasDrawBitmapLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas::drawImage][RecordingCanvasDrawImageLink]（这是连接App native层和Skia的操作，而且涉及到了跨线程）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[DisplayListData::drawImage][fDLDrawImageLink]  
反之，调用栈如下：  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[View#draw][ViewDrawLink]（注意，此时的实现者是root view）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[View#onDraw][ViewDrawLink]（注意，此时的实现者仍是root view）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[ViewGroup#dispatchDraw][ViewGroupDispatchDrawLink]（注意，此时的实现者仍是root view）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[ViewGroup#drawChild][ViewGroupDrawChildLink]（注意，此时的实现者仍是root view）  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[View#draw][ViewDrawLink]（注意，此时的实现者已经变成了root view的第一级子view，这是一个递归调用）  

最后，全部画完开始执行：  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNode#endRecording][endRecordingLink]（注意：这里的render node是root view的）  

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

## 4. mRootNode.endRecording：called by ThreadedRenderer#updateRootDisplayList

&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNode#endRecording][endRecordingLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#finishRecording][RCfinishRecordingLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RecordingCanvas#nFinishRecording][RCnFinishRecordingLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkiaRecordingCanvas::finishRecording][SKRCfinishRecordingLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkCanvas::restoreToCount][SkCRestoreToCountLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[SkCanvas::restore][SkCanvasRestoreLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNode#nSetDisplayList][nSetDisplayListLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[RenderNode::setStagingDisplayList][setStagingDisplayListLink]  

（1）window的render node进入endRecording，会使render node临时使用的android app的window的canvas进入finishRecording，此canvas又会使Skia的canvas进入restore的执行。  
（2）在beginRecording时，window的render node把RenderNode#mAvailableDisplayList赋值给canvas的SkiaRecordingCanvas#mDisplayList；到了endRecording，canvas释放了SkiaRecordingCanvas#mDisplayList后，把它反赋值给render node的RenderNode#mStagingDisplayList。  

## 5. 以简单的自定义组件为例

```java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        // 省略部分代码
        // Draw the example drawable on top of the text.
        if (mDrawable != null) {
            mDrawable.setBounds(paddingLeft, paddingTop,
                    paddingLeft + contentWidth, paddingTop + contentHeight);
            mDrawable.draw(canvas);
        }

        // Draw the text.
        canvas.drawText(mMyBtnViewText,
                paddingLeft + (contentWidth - mTextWidth) / 2,
                paddingTop + (contentHeight + mTextHeight) / 2,
                mTextPaint);
    }
```

我们再来看一个自定义组件的onDraw函数做了什么：  
（1）根据第一小节的分析，可以知道参数canvas来自root view。  
（2）自定义view不必关心当前渲染是CPU渲染还是GPU渲染。但是传下来的这个参数canvas已经决定了。如果canvas来自于[View#updateDisplayListIfDirty][updateDisplayListIfDirtyLink]，则canvas通过render node产生，onDraw回调的时候已经意味着自定义view要在GPU对应的canvas上draw了。  

## 附

### Canvas的继承关系

[RecordingCanvas][RecordingCanvasLink] -> [DisplayListCanvas][DisplayListCanvasLink] -> [BaseRecordingCanvas][BaseRecordingCanvasLink] -> [Canvas][CanvasLink] -> [BaseCanvas][BaseCanvasLink]  
[SkiaRecordingCanvas][SkiaRecordingCanvasLink] -> [SkiaCanvas][SkiaCanvasLink] -> [Canvas][NativeCanvasLink]

### 名词约定与释义

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
[SkCRestoreToCountLink]:https://cs.android.com/android/platform/superproject/+/master:external/skia/src/core/SkCanvas.cpp;l=770
[SkCanvasRestoreLink]:https://cs.android.com/android/platform/superproject/+/master:external/skia/src/core/SkCanvas.cpp;l=753
[SkiaDisplayListDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaDisplayList.h;l=143
[DisplayListDataDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RecordingCanvas.cpp;l=740

[endRecordingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RenderNode.java;l=402
[RCfinishRecordingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java;l=79
[RCnFinishRecordingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_DisplayListCanvas.cpp;l=133
[SKRCfinishRecordingLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp;l=58
[nSetDisplayListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_RenderNode.cpp;l=79
[setStagingDisplayListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RenderNode.cpp;l=77

[RecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java
[DisplayListCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/DisplayListCanvas.java;l=31
[BaseRecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/BaseRecordingCanvas.java;l=41
[CanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/Canvas.java;l=52
[BaseCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/BaseCanvas.java;l=43

[SkiaRecordingCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.h
[SkiaCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/SkiaCanvas.h
[NativeCanvasLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/hwui/Canvas.h

[updateViewTreeDisplayListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ThreadedRenderer.java;l=554
[updateDisplayListIfDirtyLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/View.java;l=21169
[ViewGroupDispatchGetDisplayListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewGroup.java;l=4467
[recreateChildDisplayListLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewGroup.java;l=4497
[BRCanvasDrawBitmapLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/BaseRecordingCanvas.java;l=67
[android_graphics_Canvas_drawBitmapLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/jni/android_graphics_Canvas.cpp;l=433
[SkiaRecordingCanvasDrawBitmapLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp;l=224
[RecordingCanvasDrawImageLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RecordingCanvas.cpp;l=950
[fDLDrawImageLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/libs/hwui/RecordingCanvas.cpp;l=950

[ViewDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/View.java;l=22321
[ViewOnDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/View.java;l=19908
[ViewGroupDispatchDrawLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewGroup.java;l=4204
[ViewGroupDrawChildLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/ViewGroup.java;l=4515
