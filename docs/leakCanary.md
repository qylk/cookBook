# LeakCanary
https://jsonchao.github.io/2019/01/06/Android%E4%B8%BB%E6%B5%81%E4%B8%89%E6%96%B9%E5%BA%93%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E5%85%AD%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Leakcanary%E6%BA%90%E7%A0%81%EF%BC%89/

## 如何检测activity内存泄漏
1. 监听生命周期  
在Android中，当一个Activity走完onDestroy生命周期后，说明该页面已经被销毁了，应该被系统GC回收。通过Application.registerActivityLifecycleCallbacks()方法注册Activity生命周期的监听，每当一个Activity页面销毁时候，获取到这个Activity去检测这个Activity是否真的被系统GC。

2. 检测  
当获取了待分析的对象后，需要确定这个对象是否产生了内存泄漏。  
通过WeakReference + ReferenceQueue来判断对象是否被系统GC回收，WeakReference 创建时，可以传入一个 ReferenceQueue 对象。当被 WeakReference 引用的对象的生命周期结束，一旦被 GC 检查到，GC将会把该对象添加到ReferenceQueue中，待ReferenceQueue处理。当 GC 过后对象一直不被加入 ReferenceQueue，它可能存在内存泄漏。  
当我们初步确定待分析对象未被GC回收时候，手动触发GC，再二次确认

3. 分析  
如果发生了内存泄漏，分析这块使用了Square的另一个开源库haha，<https://github.com/square/haha>，利用它获取当前内存中的heap堆信息的快照snapshot，然后通过待分析对象去snapshot里面去查找强引用关系。

## 使用一个简单例子说明原理
写一个activity，在onDestroy回调中检测activity是否泄漏。
```java
public class MainActivity extends Activity {
    static final Helper helper = new Helper();

    Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            //匿名内部类，导致Handler内存泄漏
            super.handleMessage(msg);
        }
    };

    @Override
    protected void onDestroy() {
        super.onDestroy();
        handler.sendEmptyMessageDelayed(1, 100000);//this will cause leak
        helper.track(this);
    }
}
```

activity检测辅助类Helper。
```java
public class Helper {
    final ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
    final HashSet<String> keys = new HashSet<>();

    public void track(Activity activity) {
        String key = UUID.randomUUID().toString();//给每一个activty对象取一个key绑定。
        keys.add(key);

        final WatchReference weakReference = new WatchReference(key, activity, referenceQueue);

        Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
            @Override
            public boolean queueIdle() {
                removeWeaklyReachableReferences();//移除所有被GC的对象的key

                if (isGone(weakReference)) {//key已被移除
                    System.out.println("activity did removed");
                } else {     //二次确认
                    callGC();//手动触发GC
                    removeWeaklyReachableReferences();//移除所有被GC的对象的key
                    if (!isGone(weakReference)) {//GC后，key被移除
                        System.out.println("activity leaked");
                    } else {                     //GC后，key未被移除，发生泄漏
                        System.out.println("activity did removed");
                        //todo 内存分析
                    }
                }
                return false;
            }
        });
    }

    boolean isGone(WatchReference reference) {
        return !keys.contains(reference.key);
    }

    void callGC() {
        Runtime.getRuntime().gc();
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            throw new AssertionError();
        }
        System.runFinalization();
    }

    void removeWeaklyReachableReferences() {
        WatchReference ref;
        while ((ref = (WatchReference) referenceQueue.poll()) != null) {
            keys.remove(ref.key);
        }
    }

    static class WatchReference extends WeakReference<Object> {
        final String key;

        public WatchReference(String key, Object referent, ReferenceQueue<? super Object> q) {
            super(referent, q);
            this.key = key;
        }
    }
}
```