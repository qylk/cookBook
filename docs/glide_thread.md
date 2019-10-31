Glide中的线程池集中使用在EngineJob上
```java
private final ExecutorService diskCacheService;
private final ExecutorService sourceService;

if (sourceService == null) {
    final int cores = Math.max(1, Runtime.getRuntime().availableProcessors());
    sourceService = new FifoPriorityThreadPoolExecutor(cores);
}
if (diskCacheService == null) {
    diskCacheService = new FifoPriorityThreadPoolExecutor(1);
}
```
* sourceService 主要用于网络下载-存储-解码这一流程，固定大小的线程池，线程数为手机处理器数量(availableProcessors)
* diskCacheService 主要用于本地缓存读取-解码这一流程，固定大小的线程池，线程数为1