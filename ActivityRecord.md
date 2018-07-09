# 一.引言
BroadcastRecord，ServiceRecord都继承于Binder对象，而ActivityRecord并没有继承于Binder。 但ActivityRecord的成员变量appToken的数据类型为Token，Token继承于IApplicationToken.Stub。

appToken：system_server进程通过调用scheduleLaunchActivity()将appToken传递到App进程，
 * 调用createActivityContext()，保存到ContextImpl.mActivityToken
 * 调用activity.attach()，保存到Activity.mToken；
 
ServiceRecord本身继承于Binder对象，传递到客户端的代理：
 * 调用Service.attach()，保存到Service.mToken；
 * 用途：stopSelf,startForeground, stopForeground

# 二. ActivityRecord结构体
![](http://gityuan.com/images/ams/activity/activity_record.jpg)

* ActivityRecord: 记录着Activity信息
* TaskRecord: 记录着task信息
* ActivityStack: 栈信息

## 2.1 ActivityRecord
Activity的信息记录在ActivityRecord对象, 并通过通过成员变量task指向TaskRecord

* ProcessRecord app //跑在哪个进程
* TaskRecord task //跑在哪个task
* ActivityInfo info // Activity信息
* int mActivityType //Activity类型
* ActivityState state //Activity状态
* ApplicationInfo appInfo //跑在哪个app
* ComponentName realActivity //组件名
* String packageName //包名
* String processName //进程名
* int launchMode //启动模式
* int userId // 该Activity运行在哪个用户id

mActivityType：

* APPLICATION_ACTIVITY_TYPE：普通应用类型
* HOME_ACTIVITY_TYPE：桌面类型
* RECENTS_ACTIVITY_TYPE：最近任务类型

ActivityState：

* INITIALIZING
* RESUMED：已恢复
* PAUSING
* PAUSED：已暂停
* STOPPING
* STOPPED：已停止
* FINISHING
* DESTROYING
* DESTROYED：已销毁


时间点	|赋值时间|	含义
--||--
createTime	    |new ActivityRecord	    |Activity首次创建时间点
displayStartTime|	AS.setLaunchTime	|Activity首次启动时间点
fullyDrawnStartTime	|AS.setLaunchTime	|Activity首次启动时间点
startTime	    | -- |	Activity上次启动的时间点
lastVisibleTime	|AR.windowsVisibleLocked	|Activity上次成为可见的时间点
cpuTimeAtResume	|AS.completeResumeLocked	|从Rsume以来的cpu使用时长
pauseTime	|AS.startPausingLocked	|Activity上次暂停的时间点
launchTickTime	|AR.startLaunchTickingLocked	|Eng版本才赋值
lastLaunchTime	|ASS.realStartActivityLocked	|上一次启动时间

其中AR是指ActivityRecord, AS是指ActivityStack。

## 2.2 TaskRecord
Task的信息记录在TaskRecord对象.

* ActivityStack stack; //当前所属的stack
* ArrayList mActivities; // 当前task的所有Activity列表
* int taskId
* String affinity； 是指root activity的affinity，即该Task中第一个Activity;
* int mCallingUid;
* String mCallingPackage； //调用者的包名

## 2.3 ActivityStack
* ArrayList mTaskHistory //保存所有的Task列表
* ArrayList mStacks; //所有stack列表
* final int mStackId;
* int mDisplayId;
* ActivityRecord mPausingActivity //正在pause
* ActivityRecord mLastPausedActivity
* ActivityRecord mResumedActivity //已经resumed
* ActivityRecord mLastStartedActivity


所有前台stack的mResumedActivity的state == RESUMED, 则表示allResumedActivitiesComplete, 此时mLastFocusedStack = mFocusedStack;

## 2.4 ActivityStackSupervisor
* ActivityStack mHomeStack //桌面的stack
* ActivityStack mFocusedStack //当前聚焦stack
* ActivityStack mLastFocusedStack //正在切换
* SparseArray mActivityDisplays //displayId为key
* SparseArray mActivityContainers // mStackId为key

home的栈ID等于0,即HOME_STACK_ID = 0;

## Stack组成图
![](http://gityuan.com/images/ams/activity/ams_relations.jpg)
* 一般地，对于没有分屏功能以及虚拟屏的情况下，ActivityStackSupervisor与ActivityDisplay都是系统唯一；
* ActivityDisplay主要有Home Stack和App Stack这两个栈；
* 每个ActivityStack中可以有若干个TaskRecord对象；
* 每个TaskRecord包含如果个ActivityRecord对象；
* 每个ActivityRecord记录一个Activity信息。

(1)正向关系链表：
```
ActivityStackSupervisor.mActivityDisplays 
-> ActivityDisplay.mStacks 
-> ActivityStack.mTaskHistory 
-> TaskRecord.mActivities 
-> ActivityRecord
```
(2)反向关系链表：
```
ActivityRecord.task 
-> TaskRecord.stack 
-> ActivityStack.mStackSupervisor
-> ActivityStackSupervisor
```
注：ActivityStack.mDisplayId可找到所对应的ActivityDisplay；

# 启动过程
![](http://gityuan.com/images/ams/activity/Seq_activity.jpg)

