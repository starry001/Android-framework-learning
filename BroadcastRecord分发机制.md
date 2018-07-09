广播在系统中以BroadcastRecord对象来记录, 该对象有几个时间相关的成员变量.
```
final class BroadcastRecord extends Binder {
    final ProcessRecord callerApp; //广播发送者所在进程
    final String callerPackage; //广播发送者所在包名
    final List receivers;   // 包括动态注册的BroadcastFilter和静态注册的ResolveInfo

    final String callerPackage; //广播发送者
    final int callingPid;   // 广播发送者pid
    final List receivers;   // 广播接收者
    int nextReceiver;  // 下一个被执行的接收者
    IBinder receiver; // 当前正在处理的接收者
    int anrCount;   //广播ANR次数

    long enqueueClockTime;  //入队列时间
    long dispatchTime;      //分发时间
    long dispatchClockTime; //分发时间
    long receiverTime;      //接收时间(首次等于dispatchClockTime)
    long finishTime;        //广播完成时间

 }
```
enqueueClockTime 伴随着 scheduleBroadcastsLocked
dispatchClockTime伴随着 deliverToRegisteredReceiverLocked
finishTime 位于 addBroadcastToHistoryLocked方法内

## 从广播发送方式可分为三类：

* 普通广播：通过Context.sendBroadcast()发送，可并行处理
* 有序广播：通过Context.sendOrderedBroadcast()发送，串行处理
* Sticky广播：通过Context.sendStickyBroadcast()发送

## 发送广播 AMS.broadcastIntentLocked
```
private final int broadcastIntentLocked(ProcessRecord callerApp, String callerPackage, Intent intent, String resolvedType, IIntentReceiver resultTo, int resultCode, String resultData, Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle options, boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {

    //step1: 设置flag
    //step2: 广播权限验证
    //step3: 处理系统相关广播
    //step4: 增加sticky广播
    //step5: 查询receivers和registeredReceivers
    //step6: 处理并行广播
    //step7: 合并registeredReceivers到receivers
    //step8: 处理串行广播

    return ActivityManager.BROADCAST_SUCCESS;
}
```
### step1: 设置flag
```
intent = new Intent(intent);
//增加该flag，则广播不会发送给已停止的package
intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);

//当没有启动完成时，不允许启动新进程
if (!mProcessesReady && (intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0) {
    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
}
userId = handleIncomingUser(callingPid, callingUid, userId,
        true, ALLOW_NON_FULL, "broadcast", callerPackage);

//检查发送广播时用户状态
if (userId != UserHandle.USER_ALL && !isUserRunningLocked(userId, false)) {
    if ((callingUid != Process.SYSTEM_UID
            || (intent.getFlags() & Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0)
            && !Intent.ACTION_SHUTDOWN.equals(intent.getAction())) {
        return ActivityManager.BROADCAST_FAILED_USER_STOPPED;
    }
}
```
这个过程最重要的工作是：
* 添加flag=FLAG_EXCLUDE_STOPPED_PACKAGES，保证已停止app不会收到该广播；
* 当系统还没有启动完成，则不允许启动新进程，，即只有动态注册receiver才能接受广播
* 当非USER_ALL广播且当前用户并没有处于Running的情况下，除非是系统升级广播或者关机广播，否则直接返回

BroadcastReceiver还有其他flag，位于Intent.java常量:
```
FLAG_RECEIVER_REGISTERED_ONLY //只允许已注册receiver接收广播
FLAG_RECEIVER_REPLACE_PENDING //新广播会替代相同广播
FLAG_RECEIVER_FOREGROUND //只允许前台receiver接收广播
FLAG_RECEIVER_NO_ABORT //对于有序广播，先接收到的receiver无权抛弃广播
FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT //Boot完成之前，只允许已注册receiver接收广播
FLAG_RECEIVER_BOOT_UPGRADE //升级模式下，允许系统准备就绪前可以发送广播
```
### step2: 广播权限验证
```
int callingAppId = UserHandle.getAppId(callingUid);
if (callingAppId == Process.SYSTEM_UID || callingAppId == Process.PHONE_UID
    || callingAppId == Process.SHELL_UID || callingAppId == Process.BLUETOOTH_UID
    || callingAppId == Process.NFC_UID || callingUid == 0) {
    //直接通过
} else if (callerApp == null || !callerApp.persistent) {
    try {
        if (AppGlobals.getPackageManager().isProtectedBroadcast(
                intent.getAction())) {
            //不允许发送给受保护的广播
            throw new SecurityException(msg);
        } else if (AppWidgetManager.ACTION_APPWIDGET_CONFIGURE.equals(intent.getAction())) {
            ...
        }
    } catch (RemoteException e) {
        return ActivityManager.BROADCAST_SUCCESS;
    }
}
```
* 对于callingAppId为SYSTEM_UID，PHONE_UID，SHELL_UID，BLUETOOTH_UID，NFC_UID之一或者callingUid == 0时都畅通无阻；
* 否则当调用者进程为空 或者非persistent进程的情况下：
  * 当发送的是受保护广播mProtectedBroadcasts(只允许系统使用)，则抛出异常；
  * 当action为ACTION_APPWIDGET_CONFIGURE时，虽然不希望该应用发送这种广播，处于兼容性考虑，限制该广播只允许发送给自己，否则抛出异常。

### step3: 处理系统相关广播
```
final String action = intent.getAction();
if (action != null) {
    switch (action) {
        case Intent.ACTION_UID_REMOVED: //uid移除
        case Intent.ACTION_PACKAGE_REMOVED: //package移除
        case Intent.ACTION_PACKAGE_ADDED: //增加package
        case Intent.ACTION_PACKAGE_CHANGED: //package改变

        case Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE: //外部设备不可用
        case Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE: //外部设备可用

        case Intent.ACTION_TIMEZONE_CHANGED: //时区改变，通知所有运行中的进程
        case Intent.ACTION_TIME_CHANGED: //时间改变，通知所有运行中的进程

        case Intent.ACTION_CLEAR_DNS_CACHE: //DNS缓存清空
        case Proxy.PROXY_CHANGE_ACTION: //网络代理改变
    }
}
```
### step4: 增加sticky广播
这个过程主要是将sticky广播增加到list，并放入mStickyBroadcasts里面。

### step5: 查询receivers和registeredReceivers
```
List receivers = null;
List<BroadcastFilter> registeredReceivers = null;
//当允许静态接收者处理该广播，则通过PKMS根据Intent查询相应的静态receivers
if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
    receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
}
if (intent.getComponent() == null) {
    if (userId == UserHandle.USER_ALL && callingUid == Process.SHELL_UID) {
        ...
    } else {
        // 查询相应的动态注册的广播
        registeredReceivers = mReceiverResolver.queryIntent(intent,
                resolvedType, false, userId);
    }
}
```
* receivers：记录着匹配当前intent的所有静态注册广播接收者；
* registeredReceivers：记录着匹配当前的所有动态注册的广播接收者。
* 根据userId来决定广播是发送给全部的接收者，还是指定的userId;
* mReceiverResolver是AMS的成员变量，记录着已注册的广播接收者的resolver

### step6: 处理并行广播
```
//用于标识是否需要用新intent替换旧的intent。
final boolean replacePending = (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;
//处理并行广播
int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
if (!ordered && NR > 0) {
    final BroadcastQueue queue = broadcastQueueForIntent(intent);
    //创建BroadcastRecord对象
    BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
            callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
            appOp, brOptions, registeredReceivers, resultTo, resultCode, resultData,
            resultExtras, ordered, sticky, false, userId);

    final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
    if (!replaced) {
        //将BroadcastRecord加入到并行广播队列[见下文]
        queue.enqueueParallelBroadcastLocked(r);
        //处理广播
        queue.scheduleBroadcastsLocked();
    }
    //动态注册的广播接收者处理完成，则会置空该变量；
    registeredReceivers = null;
    NR = 0;
}
```
广播队列中有一个成员变量mParallelBroadcasts，类型为ArrayList，记录着所有的并行广播。
```
// 并行广播,加入mParallelBroadcasts队列
public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
    mParallelBroadcasts.add(r);
    r.enqueueClockTime = System.currentTimeMillis();
}
```
### step7: 合并registeredReceivers到receivers

动态注册的registeredReceivers，全部合并到receivers，再统一按串行方式处理。
由上一步可知，如果不是ordered广播，则会在上一步已经被处理，此时的registeredReceivers为null。如果是串行广播，registeredReceivers不为null，添加到receivers中给等待一个个串行发送

### step8: 处理串行广播
```
if ((receivers != null && receivers.size() > 0)
        || resultTo != null) {
    BroadcastQueue queue = broadcastQueueForIntent(intent);
    //创建BroadcastRecord
    BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
            callerPackage, callingPid, callingUid, resolvedType,
            requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
            resultData, resultExtras, ordered, sticky, false, userId);

    boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
    if (!replaced) {
        //将BroadcastRecord加入到有序广播队列
        queue.enqueueOrderedBroadcastLocked(r);
        //处理广播
        queue.scheduleBroadcastsLocked();
    }
}
```
广播队列中有一个成员变量mOrderedBroadcasts，类型为ArrayList，记录着所有的有序广播。
```
// 串行广播 加入mOrderedBroadcasts队列
public void enqueueOrderedBroadcastLocked(BroadcastRecord r) {
    mOrderedBroadcasts.add(r);
    r.enqueueClockTime = System.currentTimeMillis();
}
```
### 处理方式：

1. Sticky广播: 广播注册过程处理AMS.registerReceiver，开始处理粘性广播，见小节[2.5];
   * 创建BroadcastRecord对象；
   * 并添加到mParallelBroadcasts队列；
   * 然后执行queue.scheduleBroadcastsLocked；
2. 并行广播： 广播发送过程处理，见小节[3.4.6]
   * 只有动态注册的mRegisteredReceivers才会并行处理；
   * 会创建BroadcastRecord对象;
   * 并添加到mParallelBroadcasts队列；
   * 然后执行queue.scheduleBroadcastsLocked;
3. 串行广播： 广播发送广播处理，见小节[3.4.8]
   * 所有静态注册的receivers以及动态注册mRegisteredReceivers合并到一张表处理；
   * 创建BroadcastRecord对象；
   * 并添加到mOrderedBroadcasts队列；
   * 然后执行queue.scheduleBroadcastsLocked；
## 处理广播: scheduleBroadcastsLocked

![](http://gityuan.com/images/ams/send_broadcast.jpg)

## 广播处理机制
* 当发送串行广播(ordered=true)的情况下：
  * 静态注册的广播接收者(receivers)，采用串行处理；
  * 动态注册的广播接收者(registeredReceivers)，采用串行处理；
* 当发送并行广播(ordered=false)的情况下：
  * 静态注册的广播接收者(receivers)，依然采用串行处理；
  * 动态注册的广播接收者(registeredReceivers)，采用并行处理；

简单来说，静态注册的receivers始终采用串行方式来处理（processNextBroadcast）； 动态注册的registeredReceivers处理方式是串行还是并行方式, 取决于广播的发送方式(processNextBroadcast)。

静态注册的广播往往其所在进程还没有创建，而进程创建相对比较耗费系统资源的操作，所以 让静态注册的广播串行化，能防止出现瞬间启动大量进程的喷井效应。

## ANR时机：
只有串行广播才需要考虑超时，因为接收者是串行处理的，前一个receiver处理慢，会影响后一个receiver；并行广播 通过一个循环一次性向所有的receiver分发广播事件，所以不存在彼此影响的问题，则没有广播超时；

   * 串行广播超时情况1：某个广播总处理时间 > 2* receiver总个数 * mTimeoutPeriod, 其中mTimeoutPeriod，前台队列默认为10s，后台队列默认为60s;
   * 串行广播超时情况2：某个receiver的执行时间超过mTimeoutPeriod；