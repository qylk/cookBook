## Handler如何实现延迟处理消息
sendEmptyMessageDelayed(int what, long delayMillis) —>  
sendMessageDelayed(Message msg, long delayMillis) —>  
sendMessageAtTime(Message msg, long uptimeMillis) —>  
enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) ->  
MessageQueue#enqueueMessage(Message msg, long when)

```java
boolean enqueueMessage(Message msg, long when) {
        //...
        synchronized (this) {
            //...

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            // 当消息队列中没有消息，或者是按时间来说是该排第一个
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked; //如果当前在阻塞状态，需要wake，即立即停止阻塞
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                // 以时间为顺序，插入到链表队列中间
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr); //唤醒线程阻塞状态
            }
        }
        return true;
    }
```
从以上代码可以看出该类的作用是把消息按时间顺序排序，并且控制线程的唤醒。

再看看next()

```java
Message next() {
        //...
        int pendingIdleHandlerCount = -1;
        int nextPollTimeoutMillis = 0; // 阻塞的时间
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            // 阻塞操作
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // 获取系统启动后，到现在的时间
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {//当遇到同步屏障以后
                    // 查找下一条异步消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }

                if (msg != null) {
                    if (now < msg.when) {
                        // 如果时间未到，设置下一轮需要阻塞等待的时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 得到消息
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // 没有消息
                    nextPollTimeoutMillis = -1;
                }
            //...
        }
    }
```

Handler获取时间的方式是调用SystemClock.uptimeMillis()，并用它和消息的里包含的时间进行对比。

综上所述：
1. 如果sendEmptyMessageDelayed发送了消息A，延时为500ms，这时消息进入队列，触发了nativePollOnce，Looper阻塞，等待下一个消息，或者是Delayed时间结束，自动唤醒；
2. 在1的前提下，紧接着又sendEmptyMessage了消息B，消息进入队列，但这时A的阻塞时间还没有到，于是把B插入到A的前面，然后调用nativeWake()方法唤醒线程
3. 唤醒之后，会重新取队列，这是B在A前面，又不需要等待，于是直接返回给Looper
4. Looper处理完该消息后，会再次调用next()方法，如果发现now大于msg.when则返回A消息，否则计算下一次该等待的时间

注意：SystemClock.uptimeMillis()，Handler是通过它来获取时间的，但uptimeMillis()是不包括休眠的时间的，所以手机如果在休眠状态下，那时间就一直不变，所以使用Handler并不能准确的延迟执行，因为它不计算休眠时间。

## Handler同步屏障
Handler中的Message可以分为两类：同步消息、异步消息。消息类型可以通过以下函数得知
```java
//Message.java
public boolean isAsynchronous() {
    return (flags & FLAG_ASYNCHRONOUS) != 0;
}
```
一般情况下这两种消息的处理方式没什么区别，只有在设置了同步屏障时才会出现差异。  
通常我们使用Handler发消息时，这些消息都是**同步消息**，如果我们想发送异步消息，那么在创建Handler时使用以下构造函数中的其中一种(async传true)
```java
public Handler(boolean async);
public Handler(Callback callback, boolean async);
public Handler(Looper looper, Callback callback, boolean async);
//Message.java
public void setAsynchronous(boolean async);
```
然后通过该Handler发送的所有消息都会变成异步消息。

同步屏障可以通过MessageQueue.postSyncBarrier函数来设置。
```java
@hide
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```
可以看到同步屏障也是一个消息，该函数仅仅是创建了一个Message对象并加入到了消息链表中。乍一看好像没什么特别的，但是这里面有一个很大的不同点是该Message没有target，这是同步屏障的重要特征，所以从代码层面上来讲，同步屏障就是一个Message，一个target字段为空的Message。

同步屏障只在Looper死循环获取待处理消息时才会起作用，也就是说同步屏障在MessageQueue.next函数中发挥着作用。
```java
Message next() {
    //...
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        //...
        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {//碰到同步屏障
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                // do while循环遍历消息链表
                // 跳出循环时，msg指向离表头最近的一个异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
        /...
        }
    }
}
```
可以看到，当设置了同步屏障之后，next函数将会忽略所有的同步消息，优先返回异步消息。换句话说就是，设置了同步屏障之后，Handler只会处理异步消息。再换句话说，同步屏障为Handler消息机制增加了一种简单的优先级机制，异步消息的优先级要高于同步消息。

设置同步屏障消息后，如不清楚，它将一直呆在消息队列中起作用。清除同步屏障可以通过MessageQueue.removeSyncBarrier函数来设置。
```java
public void removeSyncBarrier(int token);
```

Android应用框架中为了更快的响应UI刷新事件在ViewRootImpl.scheduleTraversals中使用了同步屏障。为了让mTraversalRunnable尽快被执行，在发消息之前调用MessageQueue.postSyncBarrier设置了同步屏障。
```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //设置同步障碍，确保mTraversalRunnable优先被执行
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //内部通过Handler发送了一个异步消息
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);//清除同步屏障
        performTraversals();
    }
}
```

![Handler](./assets/65.png)

* MessageQueue 从栈底到栈顶按Message.when降序排列(相同Message.when的先进栈的离栈顶更近)的后进先出的栈(MessageQueue#enqueueMessage MessageQueue#next)
* barrier的Message与普通Message的差别是target(类型是Handler)为null，只能通过MessageQueue#postSyncBarrier创建 barrier Message
* barrier的Message与普通Message以同样的规则进栈，但是却只能通过 MessageQueue#removeSyncBarrier出栈
* 每个barrier使用独立的token(记录在Message#arg1)进行区分
* 所有的异步消息如果在barrier之后，都会被延后执行，直到调用MessageQueue#removeSyncBarrier通过其token将该barrier清除。
* 当barrier在栈顶时，栈中的异步消息照常出栈不受影响

注意：Handler中的对应构造函数被隐藏，但是可以通过调用Message#setAsynchronous指定对应的Message为asynchronous的Message。  
值得一提的是，部署barrier(MessageQueue#postSyncBarrier)与清除barrier(MessageQueue#removeSyncBarrier)的相关方法都是目前还是非公开API。


## Handler的内存泄漏原因
1. 非静态Handler
```java
private Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        //匿名内部类隐式持有外部类的this，即activity
    }
};
```
当通过Handler发送消息时，消息的target字段将引用Handler实例对象，进入消息队列，这就相当于消息间接引用了activity，如果不能在acitivity销毁前处理完，将导致内存泄漏。
```java
 private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;//taget=handler
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
}
```

2. 匿名内部类runnable。
```java
 mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                //匿名内部类隐式持有外部类的this，即activity
            }
        },1000);
```

消息的callback字段将引用Runnable，而Runnable引用外部activity，也可能导致内存泄漏。
```java
 private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

解决办法：
1. 静态Handler，匿名内部类将无法引用外部activity，通过在Handler的构造方法中加入activity弱引用，以实现对activity的访问。  
2. activity销毁时，移除Handler所发送的所有未处理消息。mHandler.removeCallbacksAndMessages(null);
