### adb指令杀进程：
```
am force-stop pkgName
am force-stop --user 2 pkgName //只杀用户userId=2的相关信息
```
上述am指令会触发AM的main()方法：
```
public static void main(String[] args) {
     (new Am()).run(args); 
}
```
![](http://gityuan.com/images/process/am_force_stop.jpg)

## AMS.killPackageProcessesLocked()：
1. Process: 调用AMS.killPackageProcessesLocked()清理该package所涉及的进程;
2. Activity: 调用ASS.finishDisabledPackageActivitiesLocked()清理该package所涉及的Activity;
3. Service: 调用AS.bringDownDisabledPackageServicesLocked()清理该package所涉及的Service;
4. Provider: 调用AMS.removeDyingProviderLocked()清理该package所涉及的Provider;
5. BroadcastRecevier: 调用BQ.cleanupDisabledPackageReceiversLocked()清理该package所涉及的广播

## force-stop：
![](http://gityuan.com/images/process/force_stop.jpg)

### persistent进程的特殊待遇:

1. 进程: AMS.killPackageProcessesLocked()不杀进程
2. Service: ActiveServices.collectPackageServicesLocked()不移除不清理service
3. Provider: ProviderMap.collectPackageProvidersLocked()不收集不清理provider. 且不杀该provider所连接的client的persistent进程;

### 功能点归纳：

1. force-stop并不会杀persistent进程；
2. 当app被force-stop后，无法接收到任何普通广播，那么也就常见的监听手机网络状态的变化或者屏幕亮灭的广播来拉起进程肯定是不可行；
3. 当app被force-stop后，那么alarm闹钟一并被清理，无法实现定时响起的功能；
4. app被force-stop后，四大组件以及相关进程都被一一剪除清理，即便多进程架构的app也无法拉起自己；
5. 级联诛杀：当app通过ClassLoader加载另一个app，则会在force-stop的过程中会被级联诛杀；
生死与共：当app与另个app使用了share uid，则会在force-stop的过程，任意一方被杀则另一方也被杀，建立起生死与共的强关系。