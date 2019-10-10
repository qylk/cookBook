---
title: Android StictMode
---
StrictMode类是Android 2.3 （API 9）引入的一个工具类，可以用来帮助开发者发现代码中的一些不规范的问题，以达到提升应用响应能力的目的。

严苛模式主要有2大检测策略，一个是线程策略，即TreadPolicy，另一个是VM策略，即VmPolicy。

## ThreadPolicy线程策略检测
* 自定义的耗时调用 使用detectCustomSlowCalls()开启，针对执行比较耗时的检查
* 磁盘读取操作 使用detectDiskReads()开启，用于检查当前线程中是否有磁盘读取操作
* 磁盘写入操作 使用detectDiskWrites()开启，用于检查当前线程中是否有磁盘写入操作
* 网络操作 使用detectNetwork()开启，用于检查当前线程中是否有网络请求操作

ThreadPolicy在哪个线程设置过，就在哪个线程检测。
``` java
boolean DEV_MODE = true;

public void onCreate() {
     if (DEV_MODE) {
         StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                 .detectCustomSlowCalls() //API等级11，使用StrictMode.noteSlowCode
                 .detectDiskReads()
                 .detectDiskWrites()
                 .detectNetwork()
                 .penaltyDialog() //弹出违规提示对话框
                 .penaltyLog() //在Logcat中打印违规异常信息
                 .penaltyFlashScreen() //屏幕闪烁
                 .build())
     }
     super.onCreate()
}
```

自定义耗时检查，调用StrictMode.noteSlowCall

``` java
public class TaskExecutor {

    private static long SLOW_CALL_THRESHOLD = 500;
    public void executeTask(Runnable task) {
        long startTime = SystemClock.uptimeMillis();
        task.run();
        long cost = SystemClock.uptimeMillis() - startTime;
        if (cost > SLOW_CALL_THRESHOLD) {
            StrictMode.noteSlowCall("slowCall cost=" + cost);
        }
    }
}
```

* penaltyDeath()，当触发违规条件时，直接Crash掉当前应用程序。
* penaltyDeathOnNetwork()，当触发网络违规时，Crash掉当前应用程序。
* penaltyDialog()，触发违规时，显示对违规信息对话框。
* penaltyFlashScreen()，会造成屏幕闪烁，不过一般的设备可能没有这个功能。
* penaltyDropBox()，将违规信息记录到 dropbox 系统日志目录中（/data/system/dropbox）。
* permitCustomSlowCalls()、permitDiskReads ()、permitDiskWrites()、permitNetwork 如果你想关闭某一项检测，可以使用对应的permit*方法。

## VmPolicy虚拟机策略检测
* 检查 Activity 的内存泄露情况 使用detectActivityLeaks()开启
* 检查资源没有正确关闭时提醒 使用detectLeakedClosableObjects()开启
* 泄露的Sqlite对象 使用detectLeakedSqlLiteObjects()开启
* 注册类对象是否被正确释放，如BroadcastReceiver 或 ServiceConnection，使用detectLeakedRegistrationObjects()开启
* 检测实例数量，可以协助检查内存泄露，使用setClassInstanceLimit()开启

``` java
boolean DEV_MODE = true;

public void onCreate() {
     if (DEV_MODE) {
         StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                 .detectLeakedSqlLiteObjects()
                 .detectLeakedClosableObjects() //API等级11
                 .penaltyLog()
                 .penaltyDeath()
                 .build())
     }
     super.onCreate()
}
```

## 原理
StrictMode实现原理比较简单，就是在相关的代码点插入检测代码，根据策略做检查。

以IO操作为例，主要是通过在open，read，write，close时进行监控。libcore.io.BlockGuardOs文件就是监控的地方。

以write为例，如下进行监控：
``` java
@Override
public int pwrite(FileDescriptor fd, ByteBuffer buffer, long offset) {
     BlockGuard.getThreadPolicy().onWriteToDisk();
     return os.pwrite(fd, buffer, offset);
}
```

其中onReadFromDisk()方法的实现，代码位于StrictMode.java中
``` java
// Part of BlockGuard.Policy interface:
public void onReadFromDisk() {
     if ((mPolicyMask & DETECT_DISK_READ) == 0) {
        return;
     }
     if (tooManyViolationsThisLoop()) {
        return;
     }
     startHandlingViolationException(new DiskReadViolation());
}
```

## 注意事项
* 只在开发阶段启用StrictMode，发布应用或者release版本一定要禁用它。
* 严格模式无法监控JNI中的磁盘IO和网络请求。
* 应用中并非需要解决全部的违例情况，比如有些IO操作必须在主线程中进行。