## Glide源码分析 - engine-load方法
```java
    public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {

        final String id = fetcher.getId();
        //根据众多参数生成一个key
        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());


        //根据key，从ActiveResources缓存中读取
        EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
        if (active != null) {
            cb.onResourceReady(active);
            return null;
        }

        //根据key，从MemoryCache中读取
        EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
        if (cached != null) {
            cb.onResourceReady(cached);//通知图片资源已获取到
            return null;//找到就返回
        }

        //根据key，从已经开始的jobs中找
        EngineJob current = jobs.get(key);
        if (current != null) {
            current.addCallback(cb);//job还未结束，添加一个回调
            return new LoadStatus(cb, current);
        }

        //新建一个engineJob
        EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
        //新建一个decodeJob
        DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
                transcoder, diskCacheProvider, diskCacheStrategy, priority);
        //把engineJob和decodeJob包装进Runnable
        EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
        jobs.put(key, engineJob);//暂存engineJob
        engineJob.addCallback(cb);
        engineJob.start(runnable);//开始执行runnable

        return new LoadStatus(cb, engineJob);
    }
```
load方法比较复杂，但逻辑很清楚，我们知道Glide有内存缓存，load方法中一开始先生成一个由众多参数构成的key，根据这个key，先从内存中找，如果找到就返回，找不到就继续向下执行，最后会新建engineJob和decodeJob两个任务，通过EngineRunnable封装成一个Runnable，然后调用start执行EngineRunnable。
```java
    public void start(EngineRunnable engineRunnable) {
        this.engineRunnable = engineRunnable;
        future = diskCacheService.submit(engineRunnable);
    }
```
向diskCacheService线程池提交任务执行，所以我们来看EngineRunnable的run方法
```java
    public EngineRunnable(EngineRunnableManager manager, DecodeJob<?, ?, ?> decodeJob, Priority priority) {
        this.manager = manager;
        this.decodeJob = decodeJob;
        //注意这个stage，初始化为Stage.CACHE
        this.stage = Stage.CACHE;
        this.priority = priority;
    }

    @Override
    public void run() {
        if (isCancelled) {
            return;
        }

        Exception exception = null;
        Resource<?> resource = null;
        try {
            resource = decode();//看这里
        } catch (Exception e) {
            exception = e;
        }

        if (isCancelled) {
            if (resource != null) {
                resource.recycle();
            }
            return;
        }

        if (resource == null) {
            onLoadFailed(exception);
        } else {
            onLoadComplete(resource);
        }
    }
```
跟踪decode方法，首先判断当前stage是否是Stage.CACHE，会有不同的调用
```java
    private Resource<?> decode() throws Exception {
        if (isDecodingFromCache()) {
            return decodeFromCache();
        } else {
            return decodeFromSource();
        }
    }

    private boolean isDecodingFromCache() {
        return stage == Stage.CACHE;
    }
```
由于stage初始就是Stage.CACHE，所以上面会执行decodeFromCache，这里的cache指的是Glide的磁盘缓存，也就是先从磁盘缓存中decode。
```java
private Resource<?> decodeFromCache() throws Exception {
        Resource<?> result = null;
        try {
            //先从磁盘读transfrom过的缓存
            result = decodeJob.decodeResultFromCache();
        } catch (Exception e) {
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "Exception decoding result from cache: " + e);
            }
        }

        if (result == null) {
            //先从磁盘读原始缓存，后续需要transform
            result = decodeJob.decodeSourceFromCache();
        }
        return result;
    }
```
decodeFromCache方法通过decodeJob来实现，先是decodeResultFromCache，如果没有，再decodeSourceFromCache，再没有就返回null。

```java
    public Resource<Z> decodeResultFromCache() throws Exception {
        if (!diskCacheStrategy.cacheResult()) {//检查磁盘缓存策略
            return null;
        }
        //先取缓存
        Resource<T> transformed = loadFromCache(resultKey);
        //再transcode
        Resource<Z> result = transcode(transformed);
        return result;
    }

    private Resource<T> loadFromCache(Key key) throws IOException {
        //从磁盘缓存中找图片对应的File
        File cacheFile = diskCacheProvider.getDiskCache().get(key);
        if (cacheFile == null) {
            return null;
        }

        Resource<T> result = null;
        try {
            //从图片文件decode出结果
            result = loadProvider.getCacheDecoder().decode(cacheFile, width, height);
        } finally {
            if (result == null) {
                //发生意外，从磁盘缓存删除
                diskCacheProvider.getDiskCache().delete(key);
            }
        }
        return result;
    }    
```
再回到run方法的最后
```java
    if (resource == null) {
        onLoadFailed(exception);
    } else {
        onLoadComplete(resource);
    }
```
如果磁盘缓存里找不到，将执行onLoadFailed，否则onLoadComplete，结束任务。
```java
    private void onLoadComplete(Resource resource) {
        manager.onResourceReady(resource);
    }

    private void onLoadFailed(Exception e) {
        if (isDecodingFromCache()) {
            stage = Stage.SOURCE;
            manager.submitForSource(this);
        } else {
            manager.onException(e);
        }
    }
```
在onLoadFailed方法中，由于isDecodingFromCache成立，会将stage置为Stage.SOURCE，然后将当前任务再次提交。
```java
    @Override
    public void submitForSource(EngineRunnable runnable) {
        future = sourceService.submit(runnable);
    }
```
可以看到这次换成sourceService这个线程池来执行任务了。所以还是回去看run方法。不过这次isDecodingFromCache不成立了。
再回顾decode方法，这次将走decodeFromSource。
```java
    private Resource<?> decode() throws Exception {
        if (isDecodingFromCache()) {
            return decodeFromCache();
        } else {
            return decodeFromSource();
        }
    }
```
和decodeFromCache一样，也是由decodeJob完成。
```java
    public Resource<Z> decodeFromSource() throws Exception {
        Resource<T> decoded = decodeSource();
        return transformEncodeAndTranscode(decoded);
    }
    
    private Resource<T> decodeSource() throws Exception {
        Resource<T> decoded = null;
        try {
            final A data = fetcher.loadData(priority);//调用fetcher下载，获取到inputstream
            if (isCancelled) {
                return null;
            }
            decoded = decodeFromSourceData(data);//从inputstream解码
        } finally {
            fetcher.cleanup();
        }
        return decoded;
    }

    private Resource<T> decodeFromSourceData(A data) throws IOException {
        final Resource<T> decoded;
        if (diskCacheStrategy.cacheSource()) {//检查磁盘缓存策略
            decoded = cacheAndDecodeSourceData(data);//先把inputstream保存到磁盘缓存
        } else {
            //再进行decode
            decoded = loadProvider.getSourceDecoder().decode(data, width, height);
        }
        return decoded;
    }

    private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
        Resource<T> transformed = transform(decoded);//先对图片进行transform
        writeTransformedToCache(transformed);//把transform结果保存到磁盘缓存
        Resource<Z> result = transcode(transformed);//transcode
        return result;
    } 

    //把transform结果保存到磁盘缓存
    private void writeTransformedToCache(Resource<T> transformed) {
        if (transformed == null || !diskCacheStrategy.cacheResult()) {
            return;
        }
        SourceWriter<Resource<T>> writer = new SourceWriter<Resource<T>>(loadProvider.getEncoder(), transformed);
        diskCacheProvider.getDiskCache().put(resultKey, writer);
    }
```
经过这一系列处理，最终还是得返回一个result，我们继续跟踪onLoadComplete
```java
    private void onLoadComplete(Resource resource) {
        manager.onResourceReady(resource);
    }
```
经过线程转换后回到主线程，在handleResultOnMainThread中
```java
 private void handleResultOnMainThread() {
        //...
        engineResource = engineResourceFactory.build(resource, isCacheable);
        hasResource = true;
        //...
        for (ResourceCallback cb : cbs) {
            if (!isInIgnoredCallbacks(cb)) {
                engineResource.acquire();
                cb.onResourceReady(engineResource);
            }
        }
        // Our request is complete, so we can release the resource.
        engineResource.release();
    }
```
最后还是通过onResourceReady回调给request。
```java
    public void onResourceReady(Resource<?> resource) {
        if (resource == null) {
            onException(new Exception("Expected to receive a Resource<R> with an object of " + transcodeClass
                    + " inside, but instead got null."));
            return;
        }

        Object received = resource.get();
        if (received == null || !transcodeClass.isAssignableFrom(received.getClass())) {
            releaseResource(resource);
            onException(new Exception("Expected to receive an object of " + transcodeClass
                    + " but instead got " + (received != null ? received.getClass() : "") + "{" + received + "}"
                    + " inside Resource{" + resource + "}."
                    + (received != null ? "" : " "
                        + "To indicate failure return a null Resource object, "
                        + "rather than a Resource object containing null data.")
            ));
            return;
        }

        if (!canSetResource()) {
            releaseResource(resource);
            status = Status.COMPLETE;
            return;
        }

        //重点在这
        onResourceReady(resource, (R) received);
    }

    private void onResourceReady(Resource<?> resource, R result) {
        // We must call isFirstReadyResource before setting status.
        boolean isFirstResource = isFirstReadyResource();
        status = Status.COMPLETE;
        this.resource = resource;

        if (requestListener == null || !requestListener.onResourceReady(result, model, target, loadedFromMemoryCache,
                isFirstResource)) {
            GlideAnimation<R> animation = animationFactory.build(loadedFromMemoryCache, isFirstResource);
            //回调给target
            target.onResourceReady(result, animation);
        }
        notifyLoadSuccess();
    }    
```
我们的target是GlideDrawableImageViewTarget
```java
    @Override
    public void onResourceReady(Z resource, GlideAnimation<? super Z> glideAnimation) {
        if (glideAnimation == null || !glideAnimation.animate(resource, this)) {
            setResource(resource);
        }
    }

    @Override
    protected void setResource(GlideDrawable resource) {
        //把图片Drawable设置到ImageView上
        view.setImageDrawable(resource);
    }
```
至此，一个完整的图片加载流程结束，图片顺利渲染到ImageView上了。  

大致回顾下整个流程：
![glide](./assets/45.png)