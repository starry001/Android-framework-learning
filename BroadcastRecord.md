# 二.Broadcast数据结构
![](http://gityuan.com/images/ams/broadcast/broadcast_record.jpg)
# 2.1 BroadcastRecord
1. callerApp:广播发送者的进程；
2. ordered:是否有序广播；
3. sticky:是否粘性广播；
4. receivers:广播接收器，包括动态注册(BroadcastFilter)和静态注册(ResolveInfo)；
5. receiver：数据类型为IBinder，保存的广播所在进程的ApplicationThread对象的代理端
6. 时间点：
   * enqueueClockTime: 广播入队列时间点；
   * dispatchTime：广播分发时间点；
   * receiverTime：当前receiver开始处理时间点；
   * finishTime：广播处理完成时间点；
   
# 2.2 BroadcastFilter
只有registerReceiver()过程才会创建BroadcastFilter,也就是该对象用于动态注册的广播Receiver; BroadcastFilter继承于IntentFilter.

  * packageName:发起注册广播接收器所对应的包名;
  * owningUid:广播接收器所对应的uid;

# 2.3 ReceiverList
同样地, 只有registerReceiver()过程才会创建ReceiverList;

# AMS
![](http://gityuan.com/images/ams/broadcast/broadcast_relation1.jpg)

# 3.1 注册过程
## 动态广播
App进程通过registerReceiver()向system_server进程注册BroadcastReceiver。这个过程会将App进程的两个binder 服务对象传递给system_server进程：

  * ApplicationThread： 继承于ApplicationThreadNative, 作为ActivityThread的成员变量mAppThread；
  * InnerReceiver： 继承于IIntentReceiver.Stub，作为LoadedApk.ReceiverDispatcher的成员变量mIIntentReceiver

这里有一个ReceiverDispatcher(广播分发者)， 其个数跟BroadcastReceiver是一一对象的。每个广播接收者对应一个广播分发者， 当AMS向app发送广播时会调用到app进程的广播分发者，然后再将广播以message形式post到app的主线程，来执行onReceive().

## 静态广播
通过AndroidManifest.xml声明receiver，在系统开机过程通过PKMS来解析所有的静态广播接收者。

PKMS中有一个成员变量mReceivers，其数据类型为ActivityIntentResolver；可通过PKMS的 queryIntentReceivers()方法来查询指定Intent所对应的ResolveInfo列表。

# 3.2 发送过程
![](http://gityuan.com/images/ams/broadcast/seq_broadcast.jpg)

处理过程，根据注册方式不同执行流程略有不同。

* 根据发送的广播Intent是否带有Intent.FLAG_RECEIVER_FOREGROUND, 来决定将广播放入AMS中的前台广播队列(mFgBroadcastQueue) ,还是后台广播队列(mBgBroadcastQueue).
* 发送广播的过程, 也就是将BROADCAST_INTENT_MSG消息发送到ActivityManager线程即可返回.
* 静态注册的receiver，一定是按照order方式处理；
* 动态注册的receiver，则需要根据发送方式order来决定处理方式；即可能并行队列, 也可以在串行队列
* 位于mParallelBroadcasts中的并行广播, 一次性全部发出.
* 位于mOrderedBroadcasts中的串行广播，一次处理一个，等待上一个receive完成才继续处理；