## ANR
### 类型
ANR一般有三种类型：
1. KeyDispatchTimeout(`5 seconds`) -- 主要类型  
按键或触摸事件在特定时间内无响应Reason。  
日志关键字： <font color=red> Input event dispatching timedout</font>

2. BroadcastTimeout(`10 seconds`)  
BroadcastReceiver在特定时间内无法处理完成

3. ServiceTimeout(`20 seconds`) -- 小概率类型

## 常见原因
1. 主线程耗时操作，如复杂的layout，庞大的for循环，IO等
2. 主线程被子线程同步锁block
3. 主线程被Binder对端block
4. Binder被占满导致主线程无法和SystemServer通信
5. 得不到系统资源（CPU/RAM/IO）

## 日志
```
//发生ANR的时间和生成trace.txt的时间
04-01 13:12:11.572 I/InputDispatcher( 220): Application is not responding:Window{2b263310com.android.email/com.android.email.activity.SplitScreenActivitypaused=false}.  5009.8ms since event, 5009.5ms since waitstarted
//已经等待了5009.8ms
04-01 13:12:11.572 I/WindowManager( 220): Input event dispatching timedout sending tocom.android.email/com.android.email.activity.SplitScreenActivity

04-01 13:12:14.123 I/Process(  220): Sending signal. PID: 21404 SIG: 3
04-01 13:12:14.123 I/dalvikvm(21404):threadid=4: reacting to signal 3 
……
//发生ANR的页面和进程
04-01 13:12:15.872 E/ActivityManager(  220): ANR in com.android.email(com.android.email/.activity.SplitScreenActivity)
//anr类型
04-01 13:12:15.872 E/ActivityManager(  220): Reason:keyDispatchingTimedOut 

//Load是1分钟,5分钟,15分钟CPU的负载
04-01 13:12:15.872 E/ActivityManager(  220): Load: 8.68 / 8.37 / 8.53

//CPU在ANR发生前的使用情况
04-01 13:12:15.872 E/ActivityManager(  220):CPUusage from 4361ms to 699ms ago 

//CPU的使用率 + 用户态的使用率 + 内核态的使用率
04-01 13:12:15.872 E/ActivityManager(  220):   5.5%21404/com.android.email: 1.3% user + 4.1% kernel / faults: 10 minor
//minor:高速缓存的缺页次数, major:内存的缺页次数
04-01 13:12:15.872 E/ActivityManager(  220):   4.3%220/system_server: 2.7% user + 1.5% kernel / faults: 11 minor 2 major
04-01 13:12:15.872 E/ActivityManager(  220):   0.9%52/spi_qsd.0: 0% user + 0.9% kernel
04-01 13:12:15.872 E/ActivityManager(  220):   0.5%65/irq/170-cyttsp-: 0% user + 0.5% kernel
04-01 13:12:15.872 E/ActivityManager(  220):   0.5%296/com.android.systemui: 0.5% user + 0% kernel
//注意iowait
04-01 13:12:15.872 E/ActivityManager(  220): 100%TOTAL: 4.8% user + 7.6% kernel + 87% iowait

//ANR后CPU的使用量
04-01 13:12:15.872 E/ActivityManager(  220):CPUusage from 3697ms to 4223ms later 
04-01 13:12:15.872 E/ActivityManager(  220):   25%21404/com.android.email: 25% user + 0% kernel / faults: 191 minor
04-01 13:12:15.872 E/ActivityManager(  220):    16% 21603/__eas(par.hakan: 16% user + 0% kernel
04-01 13:12:15.872 E/ActivityManager(  220):    7.2% 21406/GC: 7.2% user + 0% kernel
04-01 13:12:15.872 E/ActivityManager(  220):    1.8% 21409/Compiler: 1.8% user + 0% kernel
04-01 13:12:15.872 E/ActivityManager(  220):   5.5%220/system_server: 0% user + 5.5% kernel / faults: 1 minor
04-01 13:12:15.872 E/ActivityManager(  220):    5.5% 263/InputDispatcher: 0% user + 5.5% kernel
04-01 13:12:15.872 E/ActivityManager(  220): 32%TOTAL: 28% user + 3.7% kernel
```
从LOG可以看出ANR的类型，CPU的使用情况。

* 如果CPU使用量接近100%，说明当前设备很忙，有可能是CPU饥饿导致了ANR
* 如果CPU使用量很少，说明主线程被BLOCK了
* 如果IOwait很高，说明ANR有可能是主线程在进行I/O操作造成的

## trace文件
除了看LOG，解决ANR还得需要trace.txt文件。
### 旧版本系统
ANR的log都是放在data/anr/traces.txt
```
adb shell cat /data/anr/traces.txt > /.../traces.txt 可以导出日志文件
```
### 新版本系统
/data/anr/目录没有traces.txt，而是anr_XXX多个文件，且使用adb shell cat命令会报错，提示permission denied。
需要使用
```
adb bugreport <zip_file>
```

### trace文件解析
```
-----pid 21404 at 2011-04-0113:12:14 -----  
Cmdline: com.android.email

DALVIK THREADS:
(mutexes: tll=0 tsl=0 tscl=0 ghl=0 hwl=0 hwll=0)
//main(线程名)、prio(线程优先级,默认是5)、tid(线程唯一标识ID)、NATIVE(线程状态)
"main" prio=5 tid=1 NATIVE
//线程组名称、线程挂起次数、用于调试的线程挂起次数、线程的Java对象地址、线程的Native对象地址
  | group="main" sCount=1 dsCount=0 obj=0x2aad2248 self=0xcf70
//sysTid是线程真正意义的tid(主线程的线程号和进程号相同)
  | sysTid=21404 nice=0 sched=0/0cgrp=[fopen-error:2] handle=1876218976
//堆栈，分析ANR最重要的信息
  atandroid.os.MessageQueue.nativePollOnce(Native Method)
  atandroid.os.MessageQueue.next(MessageQueue.java:119)
  atandroid.os.Looper.loop(Looper.java:110)
 at android.app.ActivityThread.main(ActivityThread.java:3688)
 at java.lang.reflect.Method.invokeNative(Native Method)
  atjava.lang.reflect.Method.invoke(Method.java:507)
  atcom.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:866)
 at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:624)
 at dalvik.system.NativeStart.main(Native Method)
```
## 线程状态
```
ZOMBIE              线程死亡，终止运行  
RUNNING/RUNNABLE    线程可运行或正在运行  
TIMED_WAIT          执行了带有超时参数的wait、sleep或join函数  
MONITOR             线程阻塞，等待获取对象锁  
WAIT                执行了无超时参数的wait函数  
INITIALIZING        新建，正在初始化，为其分配资源  
STARTING            新建，正在启动  
NATIVE              正在执行JNI本地函数  
VMWAIT              正在等待VM资源  
SUSPENDED           线程暂停，通常是由于GC或debug被暂停  
```

## 方法论
1. 首先分析log
2. 从trace.txt文件查看调用stack
3. 看代码
4. 仔细查看ANR的成因（iowait?block?memoryleak?）