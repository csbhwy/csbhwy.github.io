@[toc]

上一篇我们对Launcher启动的过程做了一个跟踪，跟踪到AMS通过socket向Zygote请求fork新进程之后就停止追踪了，现在我们单独写一篇，接着上一篇，当然也不仅仅是针对Launcher，对进程的fork过程做一个跟踪和了解
##### 1. Android系统启动流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200507094738750.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3djc2Jod3k=,size_16,color_FFFFFF,t_70)
##### 2. fork过程的流程图和调用堆栈
由于Zygote进程在启动时会创建Java虚拟机，因此通过fork而创建的Launcher程序进程可以在内部获取一个Java虚拟机的实例拷贝。fork采用copy-on-write机制，有些类如果不做改变，甚至都不用复制，子进程可以和父进程共享这部分数据，从而省去不少内存的占用

fork过程的架构流程如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200507094916314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3djc2Jod3k=,size_16,color_FFFFFF,t_70)
Zygote的调用栈关系如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200507095053581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3djc2Jod3k=,size_16,color_FFFFFF,t_70)
##### 3. Zygote fork过程源码跟踪
Zygote先fork出SystemServer进程，接着进入循环等待，用来接收Socket发来的消息，用来fork出其他应用进程，比如Launcher
```java
public static void main(String argv[]) {
    ...
    Runnable caller;
    ....
    if (startSystemServer) {
        //Zygote Fork出的第一个进程 SystmeServer
        Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
        if (r != null) {
            r.run();
            return;
        }
    }
    ...
    //循环等待fork出其他的应用进程，比如Launcher
    //最终通过调用processOneCommand()来进行进程的处理
    caller = zygoteServer.runSelectLoop(abiList);
    ...
    if (caller != null) {
        caller.run(); //执行返回的Runnable对象，进入子进程
    }
}

// ZygoteServer::runSelectLoop -> ZygoteConnection::processOneCommand
```
循环等待接收到消息后，调用processOneCommand处理：

```java
Runnable processOneCommand(ZygoteServer zygoteServer) {
    int pid = -1;
    ...
    //fork子进程，得到一个新的pid
    //fork子进程,采用copy on write方式，这里执行一次，会返回两次
    //pid=0 表示Zygote fork子进程成功
    //pid > 0 表示子进程的真正的PID
    pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid, parsedArgs.mGids,
            parsedArgs.mRuntimeFlags, rlimits, parsedArgs.mMountExternal, parsedArgs.mSeInfo,
            parsedArgs.mNiceName, fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
            parsedArgs.mInstructionSet, parsedArgs.mAppDataDir, parsedArgs.mTargetSdkVersion);
    ...
    if (pid == 0) {
        // in child, fork成功，第一次返回的pid = 0
        ...
        return handleChildProc(parsedArgs, descriptors, childPipeFd,
                parsedArgs.mStartChildZygote);
    } else {
        //in parent
        ...
        childPipeFd = null;
        handleParentProc(pid, descriptors, serverPipeFd);
        return null;
    }
}
```
pid为0表示fork成功，但此pid并不是fork出的子进程真正的pid，仅表示fork成功与否的标志。接下来看fork成功后执行的handleChildProc方法：

```java
private Runnable handleChildProc(ZygoteArguments parsedArgs, FileDescriptor[] descriptors,
        FileDescriptor pipeFd, boolean isZygote) {
    ...
    if (parsedArgs.mInvokeWith != null) {
        ...
        throw new IllegalStateException("WrapperInit.execApplication unexpectedly returned");
    } else {
        if (!isZygote) {
            // App进程将会调用到这里，执行目标类的main()方法
            return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                    parsedArgs.mRemainingArgs, null /* classLoader */);
        } else {
            return ZygoteInit.childZygoteInit(parsedArgs.mTargetSdkVersion,
                    parsedArgs.mRemainingArgs, null /* classLoader */);
        }
    }
}
```
ZygoteInit.zygoteInit如下：

```java
public static final Runnable zygoteInit(int targetSdkVersion, String[] argv,
        ClassLoader classLoader) {
    RuntimeInit.commonInit(); //初始化运行环境 
    ZygoteInit.nativeZygoteInit(); //启动Binder线程池 
     //调用程序入口函数  
    return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
}
```
在之前讲launcher启动流程时，有讲过，在ZygoteProcess::attemptZygoteSendArgsAndGetResult方法中，Socket连接Zygote进程时，会把之前组装的msg发给Zygote，包括前面的processClass ="android.app.ActivityThread"。而接下来的RuntimeInit.applicationInit会把之前传来的"android.app.ActivityThread" 传递给findStaticMain：

```java
protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
        ClassLoader classLoader) {
    ...
    // startClass: 如果AMS通过socket传递过来的是 ActivityThread
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```
通过反射，拿到ActivityThread的main()方法：

```java
protected static Runnable findStaticMain(String className, String[] argv,
        ClassLoader classLoader) {
    Class<?> cl;
 
    try {
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                "Missing class when invoking static main " + className,
                ex);
    }
 
    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
                "Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
                "Problem getting static main on " + className, ex);
    }
 
    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                "Main method is not public and static on " + className);
    }
    return new MethodAndArgsCaller(m, argv);
}
```
把反射得来的ActivityThread main()入口返回给ZygoteInit的main，通过caller.run()进行调用：

```java
static class MethodAndArgsCaller implements Runnable {
    /** method to call */
    private final Method mMethod;
 
    /** argument array */
    private final String[] mArgs;
 
    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }
 
    //调用ActivityThread的main()
    public void run() {
        try {
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}
```
##### 4. fork成功之后ActivityThread及后续
从上面分析可知，Zygote fork出了应用的进程，并把接下来的应用启动任务交给了ActivityThread来进行，接下来我们就从ActivityThread main()来分析应用的创建过程（以Launcher为例）

调用方法堆栈如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200507104028513.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3djc2Jod3k=,size_16,color_FFFFFF,t_70)
主线程处理， 创建ActivityThread对象，调用attach进行处理，最终进入Looper循环：

```java
public static void main(String[] args) {
    // 安装选择性的系统调用拦截
    AndroidOs.install();
  ...
  //主线程处理
    Looper.prepareMainLooper();
  ...
  
  //之前SystemServer调用attach传入的是true，这里到应用进程传入false就行
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
  ...
  //一直循环，如果退出，说明程序关闭
    Looper.loop();
 
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
调用ActivityThread的attach进行处理：

```java
private void attach(boolean system, long startSeq) {
  sCurrentActivityThread = this;
  mSystemThread = system;
  if (!system) {
    //应用进程启动，走该流程
    ...
    RuntimeInit.setApplicationObject(mAppThread.asBinder());
     //获取AMS的本地代理类
    final IActivityManager mgr = ActivityManager.getService();
    try {
      //通过Binder调用AMS的attachApplication方法
      mgr.attachApplication(mAppThread, startSeq);
    } catch (RemoteException ex) {
      throw ex.rethrowFromSystemServer();
    }
    ...
  } else {
    //通过system_server启动ActivityThread对象
    ...
  }
 
  // 为 ViewRootImpl 设置配置更新回调，
  // 当系统资源配置（如：系统字体）发生变化时，通知系统配置发生变化
  ViewRootImpl.ConfigChangedCallback configChangedCallback
      = (Configuration globalConfig) -> {
    synchronized (mResourcesManager) {
      ...
    }
  };
  ViewRootImpl.addConfigCallback(configChangedCallback);
}
```
清除一些无用的记录，最终调用ActivityStackSupervisor.java的 realStartActivityLocked()，进行Activity的启动：

```java
public final void attachApplication(IApplicationThread thread, long startSeq) {
    synchronized (this) {
        //通过Binder获取传入的pid信息
        int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid, callingUid, startSeq);
        Binder.restoreCallingIdentity(origId);
    }
}

private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid, int callingUid, long startSeq) {
  ...
    //如果当前的Application记录仍然依附到之前的进程中，则清理掉
    if (app.thread != null) {
        handleAppDiedLocked(app, true, true);
    }·
 
    //mProcessesReady这个变量在AMS的 systemReady 中被赋值为true，
    //所以这里的normalMode也为true
    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    ...
    if (normalMode) {
        ...
        //调用ATM的attachApplication()，最终层层调用到ActivityStackSupervisor.java的 realStartActivityLocked()
        didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
        ...
    }
    ...
    return true;
}

// ActivityTaskManagerInternal::attachApplication -> 
// RootActivityContainer::attachApplication -> 
// StackSupervisor::realStartActivityLocked
```
真正准备去启动Activity，通过clientTransaction.addCallback把LaunchActivityItem的obtain作为回调参数加进去，再调用ClientLifecycleManager.scheduleTransaction()得到LaunchActivityItem的execute()方法进行最终的执行。参考上面的第三阶段的调用栈流程

调用栈如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200507111309218.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3djc2Jod3k=,size_16,color_FFFFFF,t_70)

```java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
        boolean andResume, boolean checkConfig) throws RemoteException {
    // 直到所有的 onPause() 执行结束才会去启动新的 activity
    if (!mRootActivityContainer.allPausedActivitiesComplete()) {
        ...
        return false;
    }
    try {
        // Create activity launch transaction.
        // 添加 LaunchActivityItem
        final ClientTransaction clientTransaction = ClientTransaction.obtain(
                    proc.getThread(), r.appToken);
        //LaunchActivityItem.obtain(new Intent(r.intent)作为回调参数
        clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                    System.identityHashCode(r), r.info,
        // TODO: Have this take the merged configuration instead of separate global
        // and override configs.
        mergedConfiguration.getGlobalConfiguration(),
        mergedConfiguration.getOverrideConfiguration(), r.compat,
        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
        r.icicle, r.persistentState, results, newIntents,
        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                r.assistToken));
        ...
        // 设置生命周期状态
        final ActivityLifecycleItem lifecycleItem;
        if (andResume) {
            lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
        } else {
            lifecycleItem = PauseActivityItem.obtain();
        }
        clientTransaction.setLifecycleStateRequest(lifecycleItem);

        // Schedule transaction.
        // 重点关注：调用 ClientLifecycleManager.scheduleTransaction()，得到上面addCallback的LaunchActivityItem的execute()方法
        mService.getLifecycleManager().scheduleTransaction(clientTransaction);

    } catch (RemoteException e) {
        if (r.launchFailed) {
             // 第二次启动失败，finish activity
            stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                    "2nd-crash", false);
            return false;
        }
        // 第一次失败，重启进程并重试
        r.launchFailed = true;
        proc.removeActivity(r);
        throw e;
    }
} finally {
    endDeferResume();
}
...
return true;
}
```
执行之前realStartActivityLocked()中的 clientTransaction.addCallback
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200507111746803.png)

```java
//[TransactionExecutor.java] 
public void execute(ClientTransaction transaction) {
  ...
     // 执行 callBack，参考上面的调用栈，执行回调方法，
   //最终调用到ActivityThread的handleLaunchActivity()
    executeCallbacks(transaction);
 
     // 执行生命周期状态
    executeLifecycleState(transaction);
    mPendingActions.clear();
}
```
主要干了两件事，第一件：初始化WindowManagerGlobal；第二件：调用performLaunchActivity方法

```java
//[ActivityThread.java] 
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
  ...
  //初始化WindowManagerGlobal
    WindowManagerGlobal.initialize();
  ...
  //调用performLaunchActivity，来处理Activity
    final Activity a = performLaunchActivity(r, customIntent);
  ...
    return a;
}
```
获取ComponentName、Context，反射创建Activity，设置Activity的一些内容，比如主题等； 最终调用callActivityOnCreate()来执行Activity的onCreate()方法

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
     // 获取 ComponentName
    ComponentName component = r.intent.getComponent();
  ...
     // 获取 Context
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
         // 反射创建 Activity
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
    ...
    }
 
    try {
        // 获取 Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        if (activity != null) {
      ...
      //Activity的一些处理
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);
 
            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
      ...
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
              // 设置主题
                activity.setTheme(theme);
            }
 
            activity.mCalled = false;
            // 执行 onCreate()
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
      ...
            r.activity = activity;
        }
    //当前状态为ON_CREATE
        r.setState(ON_CREATE);
    ...
    } catch (SuperNotCalledException e) {
        throw e;
    } catch (Exception e) {
    ...
    }
    return activity;
}
```
callActivityOnCreate先执行activity onCreate的预处理，再去调用Activity的onCreate，最终完成Create创建后的内容处理:

```java
public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    prePerformCreate(activity); //activity onCreate的预处理
    activity.performCreate(icicle, persistentState);//执行onCreate()
    postPerformCreate(activity); //activity onCreate创建后的一些信息处理
}
```
performCreate()主要调用Activity的onCreate()

```java
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
  ...
    if (persistentState != null) {
        onCreate(icicle, persistentState);
    } else {
        onCreate(icicle);
    }
  ...
}
```
至此，看到了我们最熟悉的Activity的onCreate()，Launcher的启动完成，Launcher被真正创建起来
##### 5. 小结
看到onCreate()后，进入到我们最熟悉的Activity的入口，Launcher的启动告一段落。整个Android的启动流程，我们也完整的分析完成。

Launcher的启动经过了三个阶段：

第一个阶段：SystemServer完成启动Launcher Activity的调用

第二个阶段：Zygote()进行Launcher进程的Fork操作

第三个阶段：进入ActivityThread的main()，完成最终Launcher的onCreate操作
