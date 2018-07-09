![](http://gityuan.com/images/android-service/am/start_service.png)

##### 当App通过调用Android API方法startService()或binderService()来生成并启动服务的过程，主要是由ActivityManagerService来完成的。

1. ActivityManagerService通过Socket通信方式向Zygote进程请求生成(fork)用于承载服务的进程ActivityThread。此处讲述启动远程服务的过程，即服务运行于单独的进程中，对于运行本地服务则不需要启动服务的过程。ActivityThread是应用程序的主线程；
2. Zygote通过fork的方法，将zygote进程复制生成新的进程，并将ActivityThread相关的资源加载到新进程；
3. ActivityManagerService向新生成的ActivityThread进程，通过Binder方式发送生成服务的请求；
4. ActivityThread启动运行服务，这便于服务启动的简易过程

# 1.startService
![](http://gityuan.com/images/android-service/am/Seq_start_service.png)
![](http://gityuan.com/images/android-service/start_service/start_service_processes.jpg)
![](http://gityuan.com/images/ams/service_lifeline.jpg)

由上图可见,造成ANR可能的原因有Binder full{step 7, 12}, MessageQueue(step 10), AMS Lock (step 13).
# 2.bindService
![](http://gityuan.com/images/ams/bind_service.jpg)

说明：

1. 图中蓝色代表的是Client进程(发起端), 红色代表的是system_server进程, 黄色代表的是target进程(service所在进程);
2. Client进程: 通过getServiceDispatcher获取Client进程的匿名Binder服务端，即LoadedApk.ServiceDispatcher.InnerConnection,该对象继承于IServiceConnection.Stub； 再通过bindService调用到system_server进程;
3. system_server进程: 依次通过scheduleCreateService和scheduleBindService方法, 远程调用到target进程; 
4. target进程: 依次执行onCreate()和onBind()方法; 将onBind()方法的返回值IBinder(作为target进程的binder服务端)通过publishService传递到system_server进程;
5. system_server进程: 利用IServiceConnection代理对象向Client进程发起connected()调用, 并把target进程的onBind返回Binder对象的代理端传递到Client进程;
6. Client进程: 回调到onServiceConnection()方法, 该方法的第二个参数便是target进程的binder代理端. 到此便成功地拿到了target进程的代理, 可以畅通无阻地进行交互.