# 一.ContentProviderRecord数据结构
![](http://gityuan.com/images/ams/provider/content_provider_record.jpg)

## 1.1 ContentProviderRecord
* provider：在ActivityThread的installProvider()过程，会创建ContentProvider对象， 该对象有一个成员变量Transport，继承于ContentProviderNative对象，作为binder服务端。经过binder传递到system_server 进程的便是ContentProvider.Transport的binder代理对象， 由publishContentProviders()过程完成赋值；
* proc：记录provider所在的进程，是在publishContentProviders()过程完成赋值；
* launchingApp：记录等待provider所在进程启动，getContentProviderImpl()过程执行创建进程之后赋值；
* connections：记录该ContentProvider的所有连接信息，
  * 添加连接过程：incProviderCountLocked
  * 减少连接过程：decProviderCountLocked，removeDyingProviderLocked，cleanUpApplicationRecordLocked；
* externalProcessTokenToHandle: 数据类型为HashMap<IBinder, ExternalProcessHandle>.
  * AMS.getContentProviderExternalUnchecked()过程会添加externalProcessTokenToHandle值;
  * CPR.hasConnectionOrHandle()或hasExternalProcessHandles()都会判断该变量是否为空.

## 1.2 ContentProviderConnection
功能:连接contentProvider与请求该provider所对应的进程

* provider:目标provider所对应的ContentProviderRecord结构体；
* client:请求该provider的客户端进程；
* waiting:该连接的client进程正在等待该provider发布

## 1.3 ProcessRecord
* pubProviders: ArrayMap<String, ContentProviderRecord>
记录当前进程所有已发布的provider;
* conProviders: ArrayList
记录当前进程跟其他进程provider所建立的连接

## 1.4 AMS
* mProviderMap记录系统所有的provider信息；
* mLaunchingProviders记录当前正在启动的provider;

## 1.5 ActivityThread
* mProviderMap: 记录App端的所有provider信息；
* mProviderRefCountMap：记录App端的所有provider引用信息；

# 二. Provider使用过程
![](http://gityuan.com/images/ams/provider/Seq_provider.jpg)

