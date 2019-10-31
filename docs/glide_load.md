## Glide源码分析 - load-into方法
继续看load方法，记下了我们的url(就是mode)，设置isModelSet为true，就结束了。
```java
 public GenericRequestBuilder load(ModelType model) {
        this.model = model;
        isModelSet = true;
        return this;
    }
```
直到load方法结束，我们的图片请求都还没有开始，都是在给RequestBuilder组装参数。  
看来重点在into方法中，into方法在GenericRequestBuilder中实现的。
```java
public Target<TranscodeType> into(ImageView view) {
        Util.assertMainThread();//!!into方法只能在主线程调!!
        return into(glide.buildImageViewTarget(view, transcodeClass));
    }
```
先将ImageView和transcodeClass包装成一个ImageViewTarget，从上面可知，transcodeClass为PicassoDrawable类型。

```java
public <Z> Target<Z> buildTarget(ImageView view, Class<Z> clazz) {
        if (PicassoDrawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new GlideDrawableImageViewTarget(view);
        } else if (Bitmap.class.equals(clazz)) {
            return (Target<Z>) new BitmapImageViewTarget(view);
        } else if (Drawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new DrawableImageViewTarget(view);
        } else {
            throw new IllegalArgumentException("Unhandled class: " + clazz
                    + ", try .as*(Class).transcode(ResourceTranscoder)");
        }
    }
```
这里对不同的transcodeClass，返回不同的Target，在我们的分析中，transcodeClass为PicassoDrawable类型，将返回GlideDrawableImageViewTarget对象。
我们看下Target接口的定义，显然Target首先是个LifecycleListener，可感知界面生命周期，是不是有点像LiveData？
为什么要感知生命周期，因为gif图片在界面不可见时，应该停止动画，这里就是靠LifecycleListener实现的。
其次Target还定义了图片资源的各种回调方法、获取viewSize和记录对应的request图片请求。
```java
public interface Target<R> extends LifecycleListener {
    //开始加载
    void onLoadStarted(Drawable placeholder);
    //加载失败
    void onLoadFailed(Exception e, Drawable errorDrawable);
    //加载成功
    void onResourceReady(R resource, GlideAnimation<? super R> glideAnimation);
    //加载清除
    void onLoadCleared(Drawable placeholder);
    //获取viewSize
    void getSize(SizeReadyCallback cb);
    
    //记录对应的request
    void setRequest(Request request);
    Request getRequest();
}
```
Target接口的各种实现类，无非就是实现Target接口方法，有关ImageView的各种Target就是把ImageView对象包装成一个target。
有了target，继续看into方法
```java
public <Y extends Target<TranscodeType>> Y into(Y target) {
        Util.assertMainThread();
    
        //看有没有旧的request，有就先取消
        Request previous = target.getRequest();
        if (previous != null) {
            previous.clear();
            requestTracker.removeRequest(previous);
            previous.recycle();
        }

        //RequestBuilder终于build成request了
        Request request = buildRequest(target);
        target.setRequest(request);//request记录到target中
        lifecycle.addListener(target);//target需要监听生命周期
        requestTracker.runRequest(request);//执行request
        return target;
    }
```
先不看buildRequest方法，接着看runRequest方法，在RequestTracker类中
```java
/**
 * A class for tracking, canceling, and restarting in progress, completed, and failed requests.
 */
public class RequestTracker 
```
RequestTracker顾名思义，用于跟踪Request，可以取消，重启request的辅助类，属于RequestManager职责中的一部分。
```java
    public void runRequest(Request request) {
        requests.add(request);//加入列表
        if (!isPaused) {//RequestManager未暂停？
            request.begin();//开始
        } else {
            pendingRequests.add(request);//加入等待列表
        }
    }
```
好像加入了2个列表requests和pendingRequests，有什么区别？
看下定义：
```java
 private final Set<Request> requests = Collections.newSetFromMap(new WeakHashMap<Request, Boolean>());
 private final List<Request> pendingRequests = new ArrayList<Request>();
```
requests是弱引用列表，pendingRequests是强引用列表，所有的request都被加入了requests中，因为是弱引用，不存在内存泄漏风险。而
因为Glide暂停而未执行的request被加入pendingRequests中，纯粹是为了防止这些request对象被回收，因为Glide在恢复(resume)时还需要执行这些request。
我们上面看到在buildRequest后，request被记录到了target中，瞅一眼ViewTarget中的setRequest方法
```java
    /**
     * Stores the request using {@link View#setTag(Object)}.
     */
    @Override
    public void setRequest(Request request) {
        setTag(request);
    }
```
可以看到request通过View的setTag被View强引用，只要View不回收，request就不会被回收。但是Target并不都是ViewTarget类型，所以Glide担心开发者在Target中未强引用request对象，导致request还被未执行就被回收了，才设计了上面所说的pendingRequests强引用列表。

接着回到runRequest方法，看request.begin，begin方法定义在GenericRequest类中
```java
    @Override
    public void begin() {
        status = Status.WAITING_FOR_SIZE;
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
            onSizeReady(overrideWidth, overrideHeight);
        } else {
            target.getSize(this);
        }
        if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
            target.onLoadStarted(getPlaceholderDrawable());
        }
    }
```
首先检查两个size是否是确定值，主要指是否是正数，或者等于特定的SIZE_ORIGINAL(指原图大小)，两值默认均为-1。
```java
private int overrideHeight = -1;
private int overrideWidth = -1;

private static boolean isValidDimension(int dimen) {
        return dimen > 0 || dimen == Target.SIZE_ORIGINAL;
}
```
如果size是确定值，则走入onSizeReady方法，否则调用target的getSize方法计算size，传入回调SizeReadyCallback参数为this，也就是说size计算完以后也会回调
onSizeReady方法，最后就是调用target的onLoadStarted通知请求要开始了。
在ImageViewTarget类中找到了onLoadStarted实现，就是渲染placeholder图片。
```java
    @Override
    public void onLoadStarted(Drawable placeholder) {
        view.setImageDrawable(placeholder);
    }
```
后面我们会知道request正式开始的地方是在onSizeReady方法中，但是我们先来看之前的异步getSize方法，定位在ViewTarget类中。
```java
        public void getSize(SizeReadyCallback cb) {
            int currentWidth = getViewWidthOrParam();
            int currentHeight = getViewHeightOrParam();
            if (isSizeValid(currentWidth) && isSizeValid(currentHeight)) {
                cb.onSizeReady(currentWidth, currentHeight);
            } else {
                // We want to notify callbacks in the order they were added and we only expect one or two callbacks to
                // be added a time, so a List is a reasonable choice.
                if (!cbs.contains(cb)) {
                    cbs.add(cb);
                }
                if (layoutListener == null) {
                    final ViewTreeObserver observer = view.getViewTreeObserver();
                    layoutListener = new SizeDeterminerLayoutListener(this);
                    observer.addOnPreDrawListener(layoutListener);
                }
            }
        }
```
可知获取size，首先是从View的LayoutParams中获取，如果还是不确定，就使用ViewTreeObserver监听View的PreDraw回调，在PreDraw回调中继续尝试获取size。
可想而知，如果size如果始终无法确定，那么onSizeReady方法将不会被调用，request也将不会被执行。所以说使用Glide一般需要明确view的大小，或者通过override接口指定图片大小，否则图片可能显示不出来。

最后来看关键的onSizeReady方法
```java
    @Override
    public void onSizeReady(int width, int height) {
        if (status != Status.WAITING_FOR_SIZE) {
            return;
        }
        status = Status.RUNNING;//request状态置为运行中

        width = Math.round(sizeMultiplier * width);//确定最终尺寸
        height = Math.round(sizeMultiplier * height);

        ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
        //获取Fetcher
        final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
        //获取Transcoder
        ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();
        
        loadedFromMemoryCache = true;
        //开始load
        loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
                priority, isMemoryCacheable, diskCacheStrategy, this);
        loadedFromMemoryCache = resource != null;
    }
```
最后落脚到engine.load方法上，传入参数很多，我们的图片请求就从这儿要开始了。