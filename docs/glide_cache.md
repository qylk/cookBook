## Glide资源请求流程
![Glide](./assets/45.png)

___
Glide中有几个缓存相关的类
```java
//根据当前机器参数计算需要设置的缓存大小
MemorySizeCalculator calculator = new MemorySizeCalculator(context);
//创建 Bitmap 池
if (bitmapPool == null) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
        int size = calculator.getBitmapPoolSize();
        bitmapPool = new LruBitmapPool(size);
    } else {
        bitmapPool = new BitmapPoolAdapter();
    }
}

//创建内存缓存（Lru算法）
if (memoryCache == null) {
    memoryCache = new LruResourceCache(calculator.getMemoryCacheSize());
}

//创建磁盘缓存(Lru算法)
if (diskCacheFactory == null) {
    //默认250M磁盘容量
    diskCacheFactory = new InternalCacheDiskCacheFactory(context);
}
```
除此之外 Engine 中还有一个 `ActiveResources` 作为第一级缓存
> 在旧版本的Glide中，你可能发现 `ActiveResources` 并不是第一级缓存，而是第二级。

## ActiveResources 与 MemoryCache
ActiveResources 是第一级缓存，表示当前活动中的资源集合。
```java
private final Map<Key, WeakReference<EngineResource<?>>> activeResources;
```
ActiveResources 中通过一个 HashMap 来存储，数据保存在一个弱引用（WeakReference）中。

通过 ResourceWeakReference 将资源、资源key和 ReferenceQueue 包装起来，作为value存入 activeResources 。
```java
activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
```
ResourceWeakReference 是一个WeakReference，没什么特别的。关键是传入了一个 ReferenceQueue 对象。
```java
private static class ResourceWeakReference extends WeakReference<EngineResource<?>> {
        private final Key key;

        public ResourceWeakReference(Key key, EngineResource<?> r, ReferenceQueue<? super EngineResource<?>> q) {
            super(r, q);
            this.key = key;
        }
    }
```
我们知道对象被GC时会先放入 ReferenceQueue 中，通过 ReferenceQueue 可以跟踪那些被GC的弱引用（或者软引用、虚引用）。
```java
private ReferenceQueue<EngineResource<?>> getReferenceQueue() {
        if (resourceReferenceQueue == null) {
            resourceReferenceQueue = new ReferenceQueue<EngineResource<?>>();
            
            MessageQueue queue = Looper.myQueue();
            queue.addIdleHandler(new RefQueueIdleHandler(activeResources, resourceReferenceQueue));
        }
        return resourceReferenceQueue;
}
```
getReferenceQueue方法中，还在当前主线程事件队列中添加 IdleHandler 回调，IdleHandler 回调是指 Looper 空闲(事件队列中没有处理任务)的时候回调。
```java
private static class RefQueueIdleHandler implements MessageQueue.IdleHandler {
        private final Map<Key, WeakReference<EngineResource<?>>> activeResources;
        private final ReferenceQueue<EngineResource<?>> queue;

        public RefQueueIdleHandler(Map<Key, WeakReference<EngineResource<?>>> activeResources,
                ReferenceQueue<EngineResource<?>> queue) {
            this.activeResources = activeResources;
            this.queue = queue;
        }

        @Override
        public boolean queueIdle() {
            //闲的时候检查 ReferenceQueue 中有没有被回收的资源对象
            ResourceWeakReference ref = (ResourceWeakReference) queue.poll();
            if (ref != null) {
                //有的话就从 activeResources 移除
                activeResources.remove(ref.key);
            }
            //返回true，表示下次继续回调
            return true;
        }
}
```
可见，通过在线程空闲的时候不断检查ReferenceQueue，如果有被GC的资源对象(也就是ResourceWeakReference)，就从activeResources中移除该资源。

我们已经知道Glide查找资源的顺序，先loadFromActiveResources，再loadFromCache，看下源码：
```java
public class Engine implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {

    private final MemoryCache cache;
    private final Map<Key, WeakReference<EngineResource<?>>> activeResources;
    ...

    private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }
        EngineResource<?> cached = getEngineResourceFromCache(key);
        if (cached != null) {
            cached.acquire();
            //注意，从memory取到的资源要存入activeResources
            activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
        }
        return cached;
    }

    private EngineResource<?> getEngineResourceFromCache(Key key) {
        //注意是remove(key)，不是get(key)
        Resource<?> cached = cache.remove(key);
        final EngineResource result;
        if (cached == null) {
            result = null;
        } else if (cached instanceof EngineResource) {
            result = (EngineResource) cached;
        } else {
            result = new EngineResource(cached, true /*isCacheable*/);
        }
        return result;
    }

    private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }
        EngineResource<?> active = null;
        WeakReference<EngineResource<?>> activeRef = activeResources.get(key);
        if (activeRef != null) {
            active = activeRef.get();
            if (active != null) {
                active.acquire();
            } else {
                //资源被回收了，要移除
                activeResources.remove(key);
            }
        }
        return active;
    }

    ...
}
```
可以看到，当从MemoryCache中获取到缓存图片之后会将它从缓存中**移除**，并转移存储到activeResources当中。  
另外注意到，每次获取到缓存资源后，都会调用acquire方法，看下acquire源码。
```java
class EngineResource<Z> implements Resource<Z> {
    ...
    private int acquired;
    private boolean isRecycled;

    void acquire() {
        ++acquired;
    }

    void release() {
        if (--acquired == 0) {
            listener.onResourceReleased(key, this);
        }
    }
}
```
EngineResource中维护了一个计数器，类似引用计数，当资源需要release时，检查计数器是否为0，如果为0，表示该资源没有引用了，资源可以回收。
调用onResourceReleased，回调到Engine中
```java
    @Override
    public void onResourceReleased(Key cacheKey, EngineResource resource) {
        activeResources.remove(cacheKey);
        if (resource.isCacheable()) {
            cache.put(cacheKey, resource);
        } else {
            resourceRecycler.recycle(resource);
        }
    }
```
首先从activeResources移除，因为其不再是active资源了，然后检查resource是否可以被缓存，如果可以，就重新回到MemoryCache中，否则将被recycle掉。
___
总结下缓存的转移图：  
![Glide](./assets/50.png)

为什么要搞一个activeResource呢？   
使用 activeResources 来缓存正在使用中的图片，用来保护正在使用中的图片不会被LruCache算法回收掉。

## DiskCache
使用DiskLruCache实现，Glide默认使用250M磁盘空间，缓存目录默认为InternalCache。

Glide的磁盘缓存除了保存下载的图片文件以外，还保存transform过的图片资源。  
看下 transformEncodeAndTranscode 方法，该方法在获取原始图片资源后调用，进行 transform 和 transcode。
```java
    private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
        Resource<T> transformed = transform(decoded);
        
        //transform结果 写入磁盘缓存
        writeTransformedToCache(transformed);

        Resource<Z> result = transcode(transformed);
        
        return result;
    }
```
是否需要使用磁盘缓存transform结果，由DiskCacheStrategy决定，其中有两个布尔变量
* cacheSource：是否保存原始图片
* cacheResult：是否保存transform后的图片
```java
public enum DiskCacheStrategy {
    /** Caches with both {@link #SOURCE} and {@link #RESULT}. */
    ALL(true, true),
    /** Saves no data to cache. */
    NONE(false, false),
    /** Saves just the original data to cache. */
    SOURCE(true, false),
    /** Saves the media item after all transformations to cache. */
    RESULT(false, true);

    private final boolean cacheSource;
    private final boolean cacheResult;
    ...
}
```
在从磁盘读取缓存时，也是优先读取transform缓存，后读取原始图片缓存，因为读原始数据，仍然需要transform，比较耗时，耗内存，所以加一级transform缓存可以加速性能。
```java
    private Resource<?> decodeFromCache() throws Exception {
        Resource<?> result = null;
        try {
            //优先从磁盘读transfrom过的缓存
            result = decodeJob.decodeResultFromCache();
        } catch (Exception e) {
        }

        if (result == null) {
            //后从磁盘读原始缓存，后续需要transform
            result = decodeJob.decodeSourceFromCache();
        }
        return result;
    }
```


## BitmapPool
BitmapPool是用来复用Bitmap从而避免重复创建Bitmap而带来的内存浪费，BitmapPool的实现类是LruBitmapPool。
BitmapPool是根据Bitmap尺寸和Bitmap.Config来查找可复用的Bitmap对象，复用过程主要发生在图片的transform阶段，因为这个阶段对Bitmap的操作比较多，创建、回收Bitmap都比较频繁，因此使用BitmapPool重用Bitmap对象，可减少内存抖动。
```java
public interface BitmapPool {
   boolean put(Bitmap bitmap);
   Bitmap get(int width, int height, Bitmap.Config config);
   ...
}
```
