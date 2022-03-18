@[toc]
#### 一、前言
Android开发人员对ANR应该毫不陌生——Application Not Responding，即**应用程序未响应**。ANR的**核心原理是消息机制的调度和超时处理**，它要求主线程在限定时间内做完任务处理，比如：启动Activity/Service、发送广播、ContentProvider查询、Input事件分发等，否则就可能发生ANR引起性能问题。处理超时时，系统会认为当前主线程已经失去响应其他操作的能力。这些耗时操作包括：密集的CPU运算、大量的IO操作、复杂的media编解码、大量需要绘制的嵌套布局等等，这些都可能会降低主线程的响应能力。
>PS：会有相当一部分的ANR问题是很难分析的，有时候由于系统底层的一些影响，导致消息调度失败，出现问题的场景难以复现。这类ANR问题往往需要花费大量的时间取了解系统的一些行为，除了了解ANR机制本身外，还需要对系统子功能模块的具体实现进行分析。

#### 二、 ANR的类型
ANR大致可以分为以下四种类型：

- Service Timeout（Timeout executing service：）
>前台服务20s内未执行完成
>后台服务200s内未执行完成

- Broadcase Timeout（Timeout of broadcase：）
>前台广播在10s内未执行完成
>后台广播在60s内为执行完成

- ContentProvider Timeout（Timeout publishing content providers：）
>ContentProvider超时，这里分为两种情况：
>【1】 publish时超过CONTENT_PROVIDER_PUBLISH_TIMEOUT 10s的时间，触发timeout publishing content providers，会直接kill，这一类的启动超时，有service，process，同样会触发.
>【2】 对给定的provider进行操作时，如果通过setDetectNotResponding方法给定了超时时间，例如query时超过设定的时间会触发ANR

#### 三、臭名昭著的ANR
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201029163017796.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3djc2Jod3k=,size_16,color_FFFFFF,t_70#pic_center)
我们遇到的ANR大部分都会出现类似上面的对话框，伴随着ANR的日志同步出现、分析一些初级的ANR问题，只需要简单看看输出的日志即可，但对于一些由系统原因（比如CPU负载过高、进程死锁）引发的ANR，就需要熟悉整个ANR机制，才有可能定位出问题的原因。

首先，通过查看log日志中的**ANR in**关键字找到**AppErrors.java**中appNotResponding()负责弹出上面这个ANR对话框。这个类核心功能就是负责显示ANR的对话框还有出现crash时出现的ForceClose对话框

appNotResponding具体代码逻辑如下：
1. 如果有设置IActivityController的话，通知client端appEarlyNotResponding
2. 检测是香是下面几种情况的ANR.如果是直接忽路
3. 记录ANR出现时的1og.方便后面调试问题
>如果没有生成对应的trace文件直接发送SIGNALQUIT命令，也就是kill-3
>
4. 添加日志到Dropbox
5. 通过Handler发送Message给AMS,通知运行在UiThread的UiHandler，显示ANR对话框

```java
//frameworks/base/services/core/java/com/android/server/am/AppErrors.java

final void appNotResponding(ProcessRecord app, ActivityRecord activity,
        ActivityRecord parent, boolean aboveSystem, final String annotation) {
    ArrayList<Integer> firstPids = new ArrayList<Integer>(5);
    SparseArray<Boolean> lastPids = new SparseArray<Boolean>(20);

    //【1】如果有设置IActivityController的话，通知client端appEarlyNotResponding
    if (mService.mController != null) {
        try {
            // 0 == continue, -1 = kill process immediately
            int res = mService.mController.appEarlyNotResponding(
                    app.processName, app.pid, annotation);
            if (res < 0 && app.pid != MY_PID) {
                app.kill("anr", true);
            }
        } catch (RemoteException e) {
            mService.mController = null;
            Watchdog.getInstance().setActivityController(null);
        }
    }
    //...省略
    synchronized (mService) {
        //【2】检测是否是下面几种情况的ANR，如果是直接返回
        // A：正在关机 B：已经出现了ANR的应用 C：已经崩溃的应用 D：AM杀死的应用 E：已经死亡的应用
        // PowerManager.reboot() can block for a long time, so ignore ANRs while shutting down.
        if (mService.mShuttingDown) {
            Slog.i(TAG, "During shutdown skipping ANR: " + app + " " + annotation);
            return;
        } else if (app.notResponding) {
            Slog.i(TAG, "Skipping duplicate ANR: " + app + " " + annotation);
            return;
        } else if (app.crashing) {
            Slog.i(TAG, "Crashing app skipping ANR: " + app + " " + annotation);
            return;
        } else if (app.killedByAm) {
            Slog.i(TAG, "App already killed by AM skipping ANR: " + app + " " + annotation);
            return;
        } else if (app.killed) {
            Slog.i(TAG, "Skipping died app ANR: " + app + " " + annotation);
            return;
        }
     //...省略
     //【3】记录ANR出现时候的log，方便调试使用
    // Log the ANR to the main log.
    StringBuilder info = new StringBuilder();
    info.setLength(0);
    info.append("ANR in ").append(app.processName);
    if (activity != null && activity.shortComponentName != null) {
        info.append(" (").append(activity.shortComponentName).append(")");
    }
    info.append("\n");
    info.append("PID: ").append(app.pid).append("\n");
    if (annotation != null) {
        info.append("Reason: ").append(annotation).append("\n");
    }
    if (parent != null && parent != activity) {
        info.append("Parent: ").append(parent.shortComponentName).append("\n");
    }

    ProcessCpuTracker processCpuTracker = new ProcessCpuTracker(true);

    // don't dump native PIDs for background ANRs unless it is the process of interest
    String[] nativeProcs = null;
    if (isSilentANR) {
        for (int i = 0; i < NATIVE_STACKS_OF_INTEREST.length; i++) {
            if (NATIVE_STACKS_OF_INTEREST[i].equals(app.processName)) {
                nativeProcs = new String[] { app.processName };
                break;
            }
        }
    } else {
        nativeProcs = NATIVE_STACKS_OF_INTEREST;
    }

    int[] pids = nativeProcs == null ? null : Process.getPidsForCommands(nativeProcs);
    ArrayList<Integer> nativePids = null;

    if (pids != null) {
        nativePids = new ArrayList<Integer>(pids.length);
        for (int i : pids) {
            nativePids.add(i);
        }
    }

    // For background ANRs, don't pass the ProcessCpuTracker to
    // avoid spending 1/2 second collecting stats to rank lastPids.
    File tracesFile = ActivityManagerService.dumpStackTraces(
            true, firstPids,
            (isSilentANR) ? null : processCpuTracker,
            (isSilentANR) ? null : lastPids,
            nativePids);

    String cpuInfo = null;
    if (ActivityManagerService.MONITOR_CPU_USAGE) {
        mService.updateCpuStatsNow();
        synchronized (mService.mProcessCpuTracker) {
            cpuInfo = mService.mProcessCpuTracker.printCurrentState(anrTime);
        }
        info.append(processCpuTracker.printCurrentLoad());
        info.append(cpuInfo);
    }

    info.append(processCpuTracker.printCurrentState(anrTime));

    Slog.e(TAG, info.toString());
    //【3.1】没有生成对应的Trace文件，直接发送SIGNAL_QUIT命令，也就是kill -3
    if (tracesFile == null) {
        // There is no trace file, so dump (only) the alleged culprit's threads to the log
        Process.sendSignal(app.pid, Process.SIGNAL_QUIT);
    }
    //...省略
    //【4】添加到Dropbox
    mService.addErrorToDropBox("anr", app, app.processName, activity, parent, annotation,
            cpuInfo, tracesFile, null);

    if (mService.mController != null) {
        try {
            // 0 == show dialog, 1 = keep waiting, -1 = kill process immediately
            int res = mService.mController.appNotResponding(
                    app.processName, app.pid, info.toString());
            if (res != 0) {
                if (res < 0 && app.pid != MY_PID) {
                    app.kill("anr", true);
                } else {
                    synchronized (mService) {
                        mService.mServices.scheduleServiceTimeoutLocked(app);
                    }
                }
                return;
            }
        } catch (RemoteException e) {
            mService.mController = null;
            Watchdog.getInstance().setActivityController(null);
        }
    }

    synchronized (mService) {
        mService.mBatteryStatsService.noteProcessAnr(app.processName, app.uid);

        if (isSilentANR) {
            app.kill("bg anr", true);
            return;
        }
        //【5】发送Message给AMS,通知运行在UiThread的UiHandler，显示ANR对话框
        // Set the app's notResponding state, and look up the errorReportReceiver
        makeAppNotRespondingLocked(app,
                activity != null ? activity.shortComponentName : null,
                annotation != null ? "ANR " + annotation : "ANR",
                info.toString());

        // Bring up the infamous App Not Responding dialog
        Message msg = Message.obtain();
        msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
        msg.obj = new AppNotRespondingDialog.Data(app, activity, aboveSystem);

        mService.mUiHandler.sendMessage(msg);
    }
}
```
分析到这里，我们看一下调用AppErrors.appNotResponding的地方：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201029165445209.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3djc2Jod3k=,size_16,color_FFFFFF,t_70#pic_center)
#### 四、ANR机制

ANR机制可以分为两部分：

- **触发机制**：Android对于不同的ANR类型(Broadcast, Service,Provider, InputEvent)都有⼀套监测机制。 
- **现场保存**：在监测到ANR以后，需要显示ANR对话框、输出⽇志(发⽣ANR时的 进程函数调⽤栈、CPU、IOWait使⽤情况等，保留案发现场)。

ANR的机制设计贯穿了Android的APP，Framework，Native层：
- **App层**：UI线程/主线程的处理逻辑

```java
frameworks/base/core/java/android/app/ActivityThread.java
```
- **Framework层**： ANR机制的核⼼

```java
frameworks/base/services/core/java/com/android/server/am/Activit
yManagerService.java
frameworks/base/services/core/java/com/android/server/am/Broadca
stQueue.java
frameworks/base/services/core/java/com/android/server/am/ActiveS
ervices.java
frameworks/base/services/core/java/com/android/server/input/Inpu
tManagerService.java
frameworks/base/services/core/java/com/android/server/wm/InputMo
nitor.java
frameworks/base/core/java/android/view/InputChannel.java
frameworks/base/services/core/java/com/android/internal/os/Proce
ssCpuTracker
##JNI层
frameworks/base/services/core/jni/com_android_server_input_Input
ManagerService.cpp
```
- **Native 层**：Input系统派发机制。针对Input Dispatch类型的ANR

```java
frameworks/native/services/inputflinger/InputDispatcher.cpp
frameworks/native/services/inputflinger/InputManager.cpp
frameworks/native/services/inputflinger/InputReader.cpp
frameworks/native/services/inputflinger/InputWindow.cpp
```
通俗来讲,ANR机制核⼼思想是⼀个超时机制：
1. 设置定时器
2. 重置定时器
3. 响铃

上面我们对ANR的概念、类型、表现和机制做了个初步的介绍，下一篇我们将对四种ANR类型结合源码做具体的解读和分析
