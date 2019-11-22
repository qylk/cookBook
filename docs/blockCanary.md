## 介绍
BlockCanary是国内开发者MarkZhai开发的一套性能监控组件，它对主线程操作进行了完全透明的监控，并能输出有效的信息，帮助开发分析、定位到问题所在，迅速优化应用。

## 如何计算主线程的方法执行耗时
Looper源码片段
```java
public static void loop() {
    ...
    for (;;) {
        ...
        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        msg.target.dispatchMessage(msg);
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        ...
    }
}
```
通过给主线程的Looper设置一个Printer，打点统计dispatchMessage方法执行的时间，如果超出阀值，表示发生卡顿，则dump出各种信息，提供开发者分析性能瓶颈。
```java
    Looper.getMainLooper().setMessageLogging(mBlockCanaryCore.monitor);
```
```java
    @Override
    public void println(String x) {
        if (mStopWhenDebugging && Debug.isDebuggerConnected()) {
            return;
        }
        if (!mPrintingStarted) {
            mStartTimestamp = System.currentTimeMillis();
            mStartThreadTimestamp = SystemClock.currentThreadTimeMillis();
            mPrintingStarted = true;
            startDump();
        } else {
            final long endTime = System.currentTimeMillis();
            mPrintingStarted = false;
            if (isBlock(endTime)) {
                notifyBlockEvent(endTime);
            }
            stopDump();
        }
    }

    private void startDump() {
        if (null != BlockCanaryInternals.getInstance().stackSampler) {
            //方法栈信息采集
            BlockCanaryInternals.getInstance().stackSampler.start();
        }

        if (null != BlockCanaryInternals.getInstance().cpuSampler) {
            //cpu信息采集
            BlockCanaryInternals.getInstance().cpuSampler.start();
        }
    }

    private void stopDump() {
        if (null != BlockCanaryInternals.getInstance().stackSampler) {
            BlockCanaryInternals.getInstance().stackSampler.stop();
        }

        if (null != BlockCanaryInternals.getInstance().cpuSampler) {
            BlockCanaryInternals.getInstance().cpuSampler.stop();
        }
    }
```
## 采集方法栈
```java
//stackSampler
@Override
protected void doSample() {
        StringBuilder stringBuilder = new StringBuilder();
        //保存栈信息
        for (StackTraceElement stackTraceElement : mCurrentThread.getStackTrace()) {
            stringBuilder
                    .append(stackTraceElement.toString())
                    .append(BlockInfo.SEPARATOR);
        }

        synchronized (sStackMap) {
            if (sStackMap.size() == mMaxEntryCount && mMaxEntryCount > 0) {
                sStackMap.remove(sStackMap.keySet().iterator().next());
            }
            sStackMap.put(System.currentTimeMillis(), stringBuilder.toString());
        }
}

private Runnable mRunnable = new Runnable() {
        @Override
        public void run() {
            doSample();

            if (mShouldSample.get()) {
                //预约下次sample
                HandlerThreadFactory.getTimerThreadHandler()
                        .postDelayed(mRunnable, mSampleInterval);
            }
        }
    };

    public void start() {
        if (mShouldSample.get()) {
            return;
        }
        mShouldSample.set(true);

        HandlerThreadFactory.getTimerThreadHandler().removeCallbacks(mRunnable);
        //预约doSample，延迟celay=0.8倍的BlockThreshold
        HandlerThreadFactory.getTimerThreadHandler().postDelayed(mRunnable,
                BlockCanaryInternals.getInstance().getSampleDelay());
    }

    public void stop() {
        if (!mShouldSample.get()) {
            return;
        }
        mShouldSample.set(false);
        //取消预约
        HandlerThreadFactory.getTimerThreadHandler().removeCallbacks(mRunnable);
    }
```

初始延迟0.8 * BlockThreshold（卡顿时长阀值）后的去获取线程的堆栈信息并保存到sStackMap中，这里认为方法执行超过BlockThreshold就表示卡顿，当方法执行时间已经到了BlockThreshold的0.8倍的时候还没执行完，那么这时候就开始采集方法执行堆栈信息了，而且这里只要方法还没执行完，就会间隔mSampleInterval去再次获取函数执行堆栈信息并保存，这里之所以遥在0.8 * BlockThreshold的时候就去获取堆栈信息时为了获取到准确的堆栈信息，因为既然函数耗时已经达到0.8 * BlockThreshold了，并且函数还没执行结束，那么很大概率上会导致卡顿了，所以提前获取函数执行堆栈保证获取到造成卡顿的函数调用堆栈的正确性。

根据任务执行的起始和结束时间，获取这段时间内的所有采样过的方法栈。比如一个函数执行了3秒，那么这里会把这三秒内的所有函数执行堆栈信息都取出来，然后再封装成BlockInfo通知到外面，同时可存到文件中，到这里造成卡顿的函数执行堆栈已经采集完成。
```java
public ArrayList<String> getThreadStackEntries(long startTime, long endTime) {
        ArrayList<String> result = new ArrayList<>();
        synchronized (sStackMap) {
            for (Long entryTime : sStackMap.keySet()) {
                if (startTime < entryTime && entryTime < endTime) {
                    result.add(BlockInfo.TIME_FORMATTER.format(entryTime)
                            + BlockInfo.SEPARATOR
                            + BlockInfo.SEPARATOR
                            + sStackMap.get(entryTime));
                }
            }
        }
        return result;
    }
```

## 采集CPU
CPU使用率高可能导致任务处理不及时，可能导致函数执行一半倍挂起，需要等到下一次cpu调度后重新继续执行，这也会导致卡顿现象，所以除了采集方法栈还需要采集CPU信息。

Linux中/proc/stat文件保存CPU当前活动信息。
```
~$ cat /proc/stat
cpu  38082 627 27594 893908 12256 581 895 0 0
cpu0 22880 472 16855 430287 10617 576 661 0 0
cpu1 15202 154 10739 463620 1639 4 234 0 0
...
```
第一行有若干列，每一列显示了CPU的各种类型的耗时，所有值都是从系统启动开始累计到当前时刻。
|参数|解析|
|---|----|
|user (38082)	|处于用户态的运行时间，不包含 nice值为负进程|
|nice (627)	    |nice值为负的进程所占用的CPU时间|
system (27594)  |处于核心态的运行时间
idle (893908)	|除IO等待时间以外的其它等待时间
iowait (12256)  |IO等待时间
irq (581)	|硬中断时间
irq (581)	|软中断时间
stealstolen(0)	|一个其他的操作系统运行在虚拟环境下所花费的时间
guest(0)	|这是在Linux内核控制下为客户操作系统运行虚拟CPU所花费的时间

总的cpu时间`totalCpuTime`=各列值相加

/proc/pid/stat文件包含了某一进程所有的活动的信息。
```
~$ cat /proc/11266/stat                                               
11266 (xxxx) S 549 548 0 0 -1 1077952832 181665 14773 1569 0 14880 4370 12 10 20 0 255 0 6168634 2513281024 7724 18446744073709551615 1 1 0 0 0 0 4612 1 1073782008 0 0 0 17 0 0 0 150 0 0 0 0 0 0 0 0 0 0

其中第13~16列是有关该进程CPU占用的信息。
```
参数|解析
---|---
utime(14列)	|该任务在用户态运行的时间，单位为jiffies
stime=(15列)	|该任务在核心态运行的时间，单位为jiffies
cutime(16列)	|所有已死线程在用户态运行的时间，单位为jiffies
cstime=(17列)	|所有已死在核心态运行的时间，单位为jiffies

进程的总Cpu时间processCpuTime = utime + stime + cutime + cstime，该值包括所有线程的cpu时间。

> 参考：http://man7.org/linux/man-pages/man5/proc.5.html


```java
@Override
protected void doSample() {
    BufferedReader cpuReader = null;
    BufferedReader pidReader = null;
    //...
        cpuReader = new BufferedReader(new InputStreamReader(
                new FileInputStream("/proc/stat")), BUFFER_SIZE);
        String cpuRate = cpuReader.readLine();
        if (cpuRate == null) {
            cpuRate = "";
        }
 
        if (mPid == 0) {
            mPid = android.os.Process.myPid();
        }
        pidReader = new BufferedReader(new InputStreamReader(
                new FileInputStream("/proc/" + mPid + "/stat")), BUFFER_SIZE);
        String pidCpuRate = pidReader.readLine();
        if (pidCpuRate == null) {
            pidCpuRate = "";
        }
        //解析字符串
        parse(cpuRate, pidCpuRate);
    //...
}

private void parse(String cpuRate, String pidCpuRate) {
    String[] cpuInfoArray = cpuRate.split(" ");
    if (cpuInfoArray.length < 9) {
        return;
    }
 
    long user = Long.parseLong(cpuInfoArray[2]);
    long nice = Long.parseLong(cpuInfoArray[3]);
    long system = Long.parseLong(cpuInfoArray[4]);
    long idle = Long.parseLong(cpuInfoArray[5]);
    long ioWait = Long.parseLong(cpuInfoArray[6]);
    long total = user + nice + system + idle + ioWait
            + Long.parseLong(cpuInfoArray[7])
            + Long.parseLong(cpuInfoArray[8]);
 
    String[] pidCpuInfoList = pidCpuRate.split(" ");
    if (pidCpuInfoList.length < 17) {
        return;
    }
 
    long appCpuTime = Long.parseLong(pidCpuInfoList[13])
            + Long.parseLong(pidCpuInfoList[14])
            + Long.parseLong(pidCpuInfoList[15])
            + Long.parseLong(pidCpuInfoList[16]);
 
    if (mTotalLast != 0) {
        StringBuilder stringBuilder = new StringBuilder();
        long idleTime = idle - mIdleLast;
        long totalTime = total - mTotalLast;
 
        stringBuilder
                .append("cpu:")
                .append((totalTime - idleTime) * 100L / totalTime)
                .append("% ")
                .append("app:")
                .append((appCpuTime - mAppCpuTimeLast) * 100L / totalTime)
                .append("% ")
                .append("[")
                .append("user:").append((user - mUserLast) * 100L / totalTime)
                .append("% ")
                .append("system:").append((system - mSystemLast) * 100L / totalTime)
                .append("% ")
                .append("ioWait:").append((ioWait - mIoWaitLast) * 100L / totalTime)
                .append("% ]");
 
        synchronized (mCpuInfoEntries) {
            mCpuInfoEntries.put(System.currentTimeMillis(), stringBuilder.toString());
            if (mCpuInfoEntries.size() > MAX_ENTRY_COUNT) {
                for (Map.Entry<Long, String> entry : mCpuInfoEntries.entrySet()) {
                    Long key = entry.getKey();
                    mCpuInfoEntries.remove(key);
                    break;
                }
            }
        }
    }
    mUserLast = user;
    mSystemLast = system;
    mIdleLast = idle;
    mIoWaitLast = ioWait;
    mTotalLast = total;
 
    mAppCpuTimeLast = appCpuTime;
}
```