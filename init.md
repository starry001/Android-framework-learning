
### init->main():

1. 解析配置文件init.rc
2. 执行各个阶段，如创建zygote
3. 调用property_init初始化属性相关的资源，并通过system_property_init()启动属性服务
4. init进入无限循环，等到事件发生

![](http://gityuan.com/images/boot/init/init_oneshot.jpg)

### Zygote init：
当Zygote进程启动后, 便会执行到frameworks/base/cmds/app_process/App_main.cpp文件的main()方法. 整个调用流程:
```
App_main.main  // app_process进程
    AppRuntime.start
    AndroidRuntime.start
        AndroidRuntime.startVm  // VM create
        AndroidRuntime.startReg // 注册一些Android核心类的JNI方法
        ZygoteInit.main (首次进入Java世界)
            registerZygoteSocket
            preload
            startSystemServer
            runSelectLoop
```

      
### ZygoteInit的main()方法
    * registerZygoteSocket() 注册socket
    * preload() 预加载资源和类
    * startSystemServer() 启动系统服务
    * runSelectLoop() 循环等待事件
    
### preload（）
```
  static void preload() {
    Log.d(TAG, "begin preload");
    preloadClasses();
    preloadResources();
    preloadOpenGL();
    preloadSharedLibraries();
    WebViewFactory.prepareWebViewInZygote();
    Log.d(TAG, "end preload");
}
```

### ZygoteInit.startSystemServer()
```
private static boolean startSystemServer(String abiList, String socketName) throws MethodAndArgsCaller, RuntimeException {
    ...

    // fork子进程system_server
    pid = Zygote.forkSystemServer(
            parsedArgs.uid, parsedArgs.gid,
            parsedArgs.gids,
            parsedArgs.debugFlags,
            null,
            parsedArgs.permittedCapabilities,
            parsedArgs.effectiveCapabilities);
    ...

    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        //进入system_server进程
        handleSystemServerProcess(parsedArgs);
    }
    return true;
}
```
### ZygoteInit.handleSystemServerProcess()
```
private static void handleSystemServerProcess( ZygoteConnection.Arguments parsedArgs) throws ZygoteInit.MethodAndArgsCaller {
    ...
    if (parsedArgs.niceName != null) {
         //设置当前进程名为"system_server"
        Process.setArgV0(parsedArgs.niceName);
    }

    final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
    if (systemServerClasspath != null) {
        //执行dex优化操作,比如services.jar
        performSystemServerDexOpt(systemServerClasspath);
    }

    if (parsedArgs.invokeWith != null) {
        ...
    } else {
        ClassLoader cl = null;
        if (systemServerClasspath != null) {
            cl = new PathClassLoader(systemServerClasspath, ClassLoader.getSystemClassLoader());
            Thread.currentThread().setContextClassLoader(cl);
        }
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }
}
```
system_server进程创建PathClassLoader类加载器.

### RuntimeInit.zygoteInit
```
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) throws ZygoteInit.MethodAndArgsCaller {

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
    redirectLogStreams(); //重定向log输出

    commonInit(); // 通用的一些初始化
    nativeZygoteInit(); // zygote初始化
    applicationInit(targetSdkVersion, argv, classLoader); 
}
```
nativeZygoteInit()方法经过层层调用,会进入app_main.cpp中的onZygoteInit()方法, Binder线程池的创建也是在这个过程,如下:
```
virtual void onZygoteInit() {
    sp<ProcessState> proc = ProcessState::self();
    proc->startThreadPool(); //启动新binder线程
}
```
applicationInit()方法经过层层调用,会抛出异常ZygoteInit.MethodAndArgsCaller(m, argv), ZygoteInit.main() 会捕捉该异常, 见下文

### ZygoteInit.main()
```
public static void main(String argv[]) {
    try {
        startSystemServer(abiList, socketName); //抛出MethodAndArgsCaller异常
        ....
    } catch (MethodAndArgsCaller caller) {
        caller.run(); //此处通过反射,会调用SystemServer.main()方法 [见小节4.4]
    } catch (RuntimeException ex) {
        ...
    }
}
```

### SystemServer.main()
```
private void run() {
    if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
        Slog.w(TAG, "System clock is before 1970; setting to 1970.");
        SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
    }
    ...

    Slog.i(TAG, "Entered the Android system server!");
    EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, SystemClock.uptimeMillis());

    Looper.prepareMainLooper();// 准备主线程looper

    //加载android_servers.so库，该库包含的源码在frameworks/base/services/目录下
    System.loadLibrary("android_servers");

    //检测上次关机过程是否失败，该方法可能不会返回
    performPendingShutdown();

    createSystemContext(); //初始化系统上下文

    //创建系统服务管理
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

    //启动各种系统服务
    try {
        startBootstrapServices(); // 启动引导服务
        startCoreServices();      // 启动核心服务
        startOtherServices();     // 启动其他服务
    } catch (Throwable ex) {
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    }

    //一直循环执行
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

##总结
### 各大核心进程启动后，都会进入各种对象所相应的main()方法
进程	| 主方法
---|---
init进程	| Init.main()
zygote进程 |	ZygoteInit.main()
app_process进程	|RuntimeInit.main
system_server进程	|SystemServer.main()
app进程	|ActivityThread.main()

注意其中app_process进程是指通过/system/bin/app_process来启动的进程，且后面跟的参数不带–zygote，即并非启动zygote进程。 比如常见的有通过adb shell方式来执行am,pm等命令，便是这种方式。

关于重要进程重启的过程，会触发哪些关联进程重启名单：

zygote：触发media、netd以及子进程(包括system_server进程)重启；
system_server: 触发zygote重启;
surfaceflinger：触发zygote重启;
servicemanager: 触发zygote、healthd、media、surfaceflinger、drm重启
所以，surfaceflinger,servicemanager,zygote自身以及system_server进程被杀都会触发Zygote重启。