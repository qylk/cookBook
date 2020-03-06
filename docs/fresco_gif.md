## 记录使用Fresco图片框架加载一张GIF图发生卡死的问题

该gif图片信息
```
尺寸: 1920px * 1080px
帧数: 355
帧长：30 ms
```

经过一番源码的探究，发现问题主要出在`DropFramesFrameScheduler`这个类的逻辑上，这个类是`AnimatedDrawable2`的一个成员，而`AnimatedDrawable2`是fresco中用来承载动图的`Drawable`，就类似静态图使用`BitmapDrawable`承载这样。
`Drawable`要绘制到画布上，就是调用其`draw`方法。

```java
public void draw(Canvas canvas) {
    
    long actualRenderTimeStartMs = now();//当前时间
    long animationTimeMs = actualRenderTimeStartMs - mStartTimeMs;//当前动画执行时间

    //根据当前动画时间，找到需要绘制哪一帧
    int frameNumberToDraw = mFrameScheduler.getFrameNumberToRender(
        animationTimeMs,
        mLastFrameAnimationTimeMs);

    //绘制该帧
    boolean frameDrawn = mAnimationBackend.drawFrame(this, canvas, frameNumberToDraw);

    long targetRenderTimeForNextFrameMs = FrameScheduler.NO_NEXT_TARGET_RENDER_TIME;
    long scheduledRenderTimeForNextFrameMs = -1;
    long actualRenderTimeEnd = now();

    //计算下一帧的时间
    targetRenderTimeForNextFrameMs =
          mFrameScheduler.getTargetRenderTimeForNextFrameMs(actualRenderTimeEnd - mStartTimeMs);
    if (targetRenderTimeForNextFrameMs != FrameScheduler.NO_NEXT_TARGET_RENDER_TIME) {
        scheduledRenderTimeForNextFrameMs =
            targetRenderTimeForNextFrameMs + mFrameSchedulingDelayMs;
        //计划在下一帧的时间是刷新
        scheduleNextFrame(scheduledRenderTimeForNextFrameMs);
    }
    //...
  }
```

另外有两个逻辑：根据时间定位帧 和 计算下一帧时间，他们都跟mFrameScheduler有关系。
`mFrameScheduler`是哪个类的对象实例呢? 在`AnimatedDrawable2`中找到答案是`DropFramesFrameScheduler`：
```java
  private static FrameScheduler createSchedulerForBackendAndDelayMethod(
      @Nullable AnimationBackend animationBackend) {
    if (animationBackend == null) {
      return null;
    }
    return new DropFramesFrameScheduler(animationBackend);
  }
```
看下`getFrameNumberToRender`方法:
```java
  @Override
  public int getFrameNumberToRender(long animationTimeMs, long lastFrameTimeMs) {
    //...
    long timeInCurrentLoopMs = animationTimeMs % getLoopDurationMs();//当前循环内的动画时间
    return getFrameNumberWithinLoop(timeInCurrentLoopMs);
  }

  int getFrameNumberWithinLoop(long timeInCurrentLoopMs) {
    int frame = 0;
    long currentDuration = 0;
    do {
      currentDuration += mAnimationInformation.getFrameDurationMs(frame);
      frame++;
      //while循环逐个累加每一帧的时间，找到当前应该在哪一帧处
    } while (timeInCurrentLoopMs >= currentDuration);
    return frame - 1;
  }
```
看到这个while循环，我顿时感觉这效率不高啊，每画一帧(或刷新)，都需要来上这么一个while循环，当帧数很大的时候它不慢吗？时间复杂度O(N/2)。
但是这里还不至于导致UI卡死。

再来看`getTargetRenderTimeForNextFrameMs`方法：
```java
@Override
  public long getTargetRenderTimeForNextFrameMs(long animationTimeMs) {
    long loopDurationMs = getLoopDurationMs();
    //...

    // The animation time in the current loop
    long timePassedInCurrentLoopMs = animationTimeMs % loopDurationMs;//当前动画时间
    // The animation time in the current loop for the next frame
    long timeOfNextFrameInLoopMs = 0;

    int frameCount = mAnimationInformation.getFrameCount();
    for (int i = 0; i < frameCount && timeOfNextFrameInLoopMs <= timePassedInCurrentLoopMs; i++) {
        //又来一个for循环遍历下一帧的时间在哪
        timeOfNextFrameInLoopMs += mAnimationInformation.getFrameDurationMs(i);
    }

    // Difference between current time in loop and next frame in loop
    long timeUntilNextFrameInLoopMs = timeOfNextFrameInLoopMs - timePassedInCurrentLoopMs;
    // Add the difference to the current animation time
    return animationTimeMs + timeUntilNextFrameInLoopMs;//返回下一帧的未来时间
  }
```
除了个for循环，也没看到有什么性能问题。

最后只能来看`drawFrame`方法，它在`mAnimationBackend`中，而`mAnimationBackend`实际上是`BitmapAnimationBackend`。

```java
  //BitmapAnimationBackend
  @Override
  public boolean drawFrame(
      Drawable parent,
      Canvas canvas,
      int frameNumber) {
    //...
    boolean drawn = drawFrameOrFallback(canvas, frameNumber, FRAME_TYPE_CACHED);
    //...
    return drawn;
  }

  private boolean drawFrameOrFallback(Canvas canvas, int frameNumber, @FrameType int frameType) {
    //...
    try {
      switch (frameType) {
        //...
        case FRAME_TYPE_CREATED:
          //先产生一张空白底图
          bitmapReference =
                mPlatformBitmapFactory.createBitmap(mBitmapWidth, mBitmapHeight, mBitmapConfig);
          //renderFrameInBitmap：把帧画到底图上
          drawn = renderFrameInBitmap(frameNumber, bitmapReference) &&
          //drawBitmapAndCache：把底图画到画布上 并内存缓存
              drawBitmapAndCache(frameNumber, bitmapReference, canvas, FRAME_TYPE_CREATED);
          break;
        //...
        default:
          return false;
      }
    } finally {
      CloseableReference.closeSafely(bitmapReference);
    }
    //...
  }
```
重点看下上面的`renderFrameInBitmap`方法。
```java
  private boolean renderFrameInBitmap(int frameNumber,
          @Nullable CloseableReference<Bitmap> targetBitmap) {
    
    // Render the image
    boolean frameRendered =
        mBitmapFrameRenderer.renderFrame(frameNumber, targetBitmap.get());
    
    return frameRendered;
  }
```
继续跟踪renderFrame方法，实现在`AnimatedDrawableBackendFrameRenderer`类中
```java
  @Override
  public boolean renderFrame(int frameNumber, Bitmap targetBitmap) {
    mAnimatedImageCompositor.renderFrame(frameNumber, targetBitmap);
    return true;
  }
```
又把任务交给了`AnimatedImageCompositor`，继续跟踪
```java
public void renderFrame(int frameNumber, Bitmap bitmap) {
    Canvas canvas = new Canvas(bitmap);
    canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.SRC);

    // If blending is required, prepare the canvas with the nearest cached frame.
    int nextIndex;
    if (!isKeyFrame(frameNumber)) {
      // Blending is required. nextIndex points to the next index to render onto the canvas.
      nextIndex = prepareCanvasWithClosestCachedFrame(frameNumber - 1, canvas);
    } else {
      // Blending isn't required. Start at the frame we're trying to render.
      nextIndex = frameNumber;
    }

    // Iterate from nextIndex to the frame number just preceding the one we're trying to render
    // and composite them in order according to the Disposal Method.
    for (int index = nextIndex; index < frameNumber; index++) {
      AnimatedDrawableFrameInfo frameInfo = mAnimatedDrawableBackend.getFrameInfo(index);
      DisposalMethod disposalMethod = frameInfo.disposalMethod;
      if (disposalMethod == DisposalMethod.DISPOSE_TO_PREVIOUS) {
        continue;
      }
      if (frameInfo.blendOperation == BlendOperation.NO_BLEND) {
        disposeToBackground(canvas, frameInfo);
      }
      //！！for循环在renderFrame！！
      mAnimatedDrawableBackend.renderFrame(index, canvas);
      mCallback.onIntermediateResult(index, bitmap);
      if (disposalMethod == DisposalMethod.DISPOSE_TO_BACKGROUND) {
        disposeToBackground(canvas, frameInfo);
      }
    }

    AnimatedDrawableFrameInfo frameInfo = mAnimatedDrawableBackend.getFrameInfo(frameNumber);
    if (frameInfo.blendOperation == BlendOperation.NO_BLEND) {
      disposeToBackground(canvas, frameInfo);
    }
    // Finally, we render the current frame. We don't dispose it.
    mAnimatedDrawableBackend.renderFrame(frameNumber, canvas);
  }
```
从上面的代码注释看，这个方法的逻辑是这样的：
1. 判断是否为关键帧，如果不是，则需要`Blend`，从内存缓存中找当前帧前面离当前帧最近的一帧缓存。  
2. 从缓存帧到当前帧的前一帧，使用for循环绘制每一帧，绘制调用的是`mAnimatedDrawableBackend.renderFrame(index, canvas)`,这个实现在`AnimatedDrawableBackendImpl`中。  
3. 根据帧的`disposalMethod`和`blendOperation`参数，决定是否调用disposeToBackground，这个是指帧绘制前后是否清除背景，也就是说有的gif图并不是帧独立的，而是顺序画出每一帧才展示出最终的完整图像，当中每一帧可能是增量的。  
4. 最后把当前帧画出来。


## 结论
结合前面`FrameScheduler`的逻辑，发现出现卡死的过程是这样的：
1. 绘制帧0的时间超过了gif帧规定的时间30ms。  
2. 当准备绘制下一帧的时候，动画时间已经超过30ms，比如已经到了70ms时间，此时按`FrameScheduler`找下一帧的逻辑，应该是要绘制帧3，此时掉了2帧没绘制，当然绘制帧3依然比较耗时，超过了30ms。意味着后面会继续掉帧。  
3. 当在绘制帧3的时候，根据`AnimatedImageCompositor`中的逻辑，前一帧 帧2 没有绘制过，也就是没有内存缓存，此时只有帧0有缓存，这时需要使用for循环从1到2顺序绘制帧1 帧2，然后再绘制帧3。  
4. 可以看出绘制帧3的时间是绘制帧0的3倍，可以想象下一帧，比如帧15的时候，需要for循环从3到14顺序绘制前面丢掉的帧，这将导致后续一段时间内掉帧率逐渐恶化，单次刷新时间逐渐增加，当时间增长到秒级时，就导致了ui卡死现象。


## 解决方案
1. 我们发现`AnimatedImageCompositor`这个类的逻辑不能动，因为它的逻辑是对的，因为gif帧可能是`不独立`的，顺序绘制每一帧而不掉帧才能还原gif完整图像。而具体绘制哪一帧`frameNumber`这个参数最初来源于`FrameScheduler`的算法。`DropFramesFrameScheduler`中根据时间来找下一帧，而不总是自增的，所以如果能修改`FrameScheduler`找下一帧的逻辑，让下一帧始终自增，这样后面`AnimatedImageCompositor`将始终只需要绘制指定帧，因为没有掉帧，这就保证了单次刷新时间总是稳定的，不会出现卡死UI的问题。

2. 想法缩小gif帧的尺寸，这样会较少每一帧的绘制时间，这个需要在解码Gif的逻辑处修改，可能需要hook `GifImage`和`GifFrame`中获取尺寸的方法，因为gif帧Bitmap的大小并不依赖View的大小。