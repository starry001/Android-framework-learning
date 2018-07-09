## 类关系图

![](http://gityuan.com/images/context/context.jpg)

1. ContextImpl:
* Application/Activity/Service通过attach() 调用父类ContextWrapper的attachBaseContext(), 从而设置父类成员变量mBase为ContextImpl对象;
* ContextWrapper的核心工作都是交给mBase(即ContextImpl)来完成;

1. Application: 四大组件属于某一Application, 获取所在Application:
* Activity/Service: 是通过调用其方法getApplication(),可主动获取当前所在
mApplication;
mApplication是由LoadedApk.makeApplication()过程所初始化的;
* Receiver: 是通过其方法onReceive()的第一个参数指向通当前所在Application,也就是只有接收到广播的时候才能拿到当前的Application对象;
* provider: 目前没有提供直接获取当前所在Application的方法, 但可通过getContext()可以获取当前的ContextImpl.

## 初始化过程会创建哪些对象
类型	| LoadedApk |	ContextImpl	| Application |	创建相应对象 |	回调方法
---|---
Activity	| √	| √	| √	| Activity	| onCreate
Service     | √ | √	| √	| Service	| onCreate
Receiver    | √ | √	| √	| BroadcastReceiver	|onReceive
Provider	| √ | √ | × | ContentProvider	|onCreate
Application	| √ | √ | √ | Application	|onCreate

唯有Provider在初始化过程并不会去创建所相应的Application对象.也就意味着当有多个Apk运行在同一个进程的情况下, 第二个apk通过Provider初始化过程再调用getContext().getApplicationContext()返回的并非Application对象, 而是NULL. 这里要注意会抛出空指针异常.

## Context attach过程
1. Application:
  * 调用attachBaseContext()将新创建ContextImpl赋值到父类ContextWrapper.mBase变量;
  * 可通过getBaseContext()获取该ContextImpl;
2. Activity/Service:
  * 调用attachBaseContext() 将新创建ContextImpl赋值到父类ContextWrapper.mBase变量;
  * 可通过getBaseContext()获取该ContextImpl;
  * 可通过getApplication()获取其所在的Application对象;
3. ContentProvider:
  * 调用attachInfo()将新创建ContextImpl保存到ContentProvider.mContext变量;
可通过getContext()获取该ContextImpl;
4. BroadcastReceiver:
  * 在onCreate过程通过参数将ReceiverRestrictedContext传递过去的.
5. ContextImpl:
  * 可通过getApplicationContext()获取Application

## getApplicationContext
绝大多数情况下, getApplication()和getApplicationContext()这两个方法完全一致, 返回值也相同; 那么两者到底有什么区别呢? 真正理解这个问题的人非常少. 接下来彻底地回答下这个问题:

getApplicationContext()这个的存在是Android历史原因. 我们都知道getApplication()只存在于Activity和Service对象; 那么对于BroadcastReceiver和ContentProvider却无法获取Application, 这时就需要一个能在Context上下文直接使用的方法, 那便是getApplicationContext().

两者对比:

1. 对于Activity/Service来说, getApplication()和getApplicationContext()的返回值完全相同; 除非厂商修改过接口;
2. BroadcastReceiver在onReceive的过程, 能使用getBaseContext().getApplicationContext获取所在Application, 而无法使用getApplication;
3. ContentProvider能使用getContext().getApplicationContext()获取所在Application. 绝大多数情况下没有问题, 但是有可能会出现空指针的问题, 情况如下:

当同一个进程有多个apk的情况下, 对于第二个apk是由provider方式拉起的, 前面介绍过provider创建过程并不会初始化所在application, 此时执行 getContext().getApplicationContext()返回的结果便是NULL. 所以对于这种情况要做好判空.

Tips: 如果对于Application理解不够深刻, 建议getApplicationContext()方法谨慎使用, 做好是否为空的判定,防止出现空指针异常.