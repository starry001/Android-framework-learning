![](http://gityuan.com/images/input/input_class.jpg)

* InputManagerService位于Java层的InputManagerService.java文件；
  * 其成员mPtr指向Native层的NativeInputManager对象；
* NativeInputManager位于Native层的com_android_server_input_InputManagerService.cpp文件；
  * 其成员mServiceObj指向Java层的IMS对象；
  * 其成员mLooper是指“android.display”线程的Looper;
* InputManager位于libinputflinger中的InputManager.cpp文件；
  * InputDispatcher和InputReader的成员变量mPolicy都是指NativeInputManager对象;
  * InputReader的成员mQueuedListener，数据类型为QueuedInputListener；通过其内部成员变量mInnerListener指向InputDispatcher对象； 这便是InputReader跟InputDispatcher交互的中间枢纽。

## 启动调用栈
```
InputManagerService(初始化)
    nativeInit
        NativeInputManager
            EventHub
            InputManager
                InputDispatcher
                    Looper
                InputReader
                    QueuedInputListener
                InputReaderThread
                InputDispatcherThread
IMS.start(启动)
    nativeStart
        InputManager.start
            InputReaderThread->run
            InputDispatcherThread->run
```

