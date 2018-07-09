![](http://gityuan.com/images/binder/binder_start_service/binder_ipc_arch.jpg)

## Binder对象的架构图：
![](http://gityuan.com/images/binder/binder_start_service/ams_ipc.jpg)

## 通信流程
![](http://gityuan.com/images/binder/binder_start_service/binder_ipc_process.jpg)

#### 图解
1. 发起端线程向Binder Driver发起binder ioctl请求后, 便采用环不断talkWithDriver,此时该线程处于阻塞状态, 直到收到如下BR_XXX命令才会结束该过程.
BR_TRANSACTION_COMPLETE: oneway模式下,收到该命令则退出
BR_REPLY: 非oneway模式下,收到该命令才退出;
BR_DEAD_REPLY: 目标进程/线程/binder实体为空, 以及释放正在等待reply的binder thread或者binder buffer;
BR_FAILED_REPLY: 情况较多,比如非法handle, 错误事务栈, security, 内存不足, buffer不足, 数据拷贝失败, 节点创建失败, 各种不匹配等问题
BR_ACQUIRE_RESULT: 目前未使用的协议;
2. 左图中waitForResponse收到BR_TRANSACTION_COMPLETE,则直接退出循环, 则没有机会执行executeCommand()方法, 故将其颜色画为灰色. 除以上5种BR_XXX命令, 当收到其他BR命令,则都会执行executeCommand过程.
3. 目标Binder线程创建后, 便进入joinThreadPool()方法, 采用循环不断地循环执行getAndExecuteCommand()方法, 当bwr的读写buffer都没有数据时,则阻塞在binder_thread_read的wait_event过程. 另外,正常情况下binder线程一旦创建则不会退出

## 非oneway
![](http://gityuan.com/images/binder/binder_start_service/binder_transaction.jpg)

1. Binder客户端或者服务端向Binder Driver发送的命令都是以BC_开头,例如本文的BC_TRANSACTION和BC_REPLY, 所有Binder Driver向Binder客户端或者服务端发送的命令则都是以BR_开头, 例如本文中的BR_TRANSACTION和BR_REPLY.
2. 只有当BC_TRANSACTION或者BC_REPLY时, 才调用binder_transaction()来处理事务. 并且都会回应调用者一个BINDER_WORK_TRANSACTION_COMPLETE事务, 经过binder_thread_read()会转变成BR_TRANSACTION_COMPLETE.
3. startService过程便是一个非oneway的过程, 那么oneway的通信过程如下所述

## oneway
![](http://gityuan.com/images/binder/binder_start_service/binder_transaction_oneway.jpg)

oneway与非oneway: 都是需要等待Binder Driver的回应消息BR_TRANSACTION_COMPLETE. 主要区别在于oneway的通信收到BR_TRANSACTION_COMPLETE则返回,而不会再等待BR_REPLY消息的到来. 另外，oneway的binder IPC则接收端无法获取对方的pid.

## 数据流
![](http://gityuan.com/images/binder/binder_transaction_data.jpg)
