
* -Xms20m  最小堆
* -Xmx30m  最大堆
* -XX:+PrintDumpOnOutOfMermoryError  OOM时dump出当前堆内存转储快照
* -Xoss128k 设置本地方法栈大小（hotspot不区分本地方法栈和虚拟机栈，用-Xss）
* -XX:PermSize  -XX:MaxPermSize  设置方法区大小
* -XX:MaxDirectMemorySize=10m 本机native内存大小，不指定的话默认为最大堆大小
* -XX:+TraceClassLoading    查看类加载信息
* -XX:+TraceClassUnLoading  查看类卸载信息
* -Xint  只使用解释器模式，编辑器不介入
* -XComp 强制优先运行编译模式，但解释器会在无法编译的情况下介入
* -XX:+TieredCompilation 开启分层编译
* -XX:UseCounterDecay 关闭热度衰减
* -XX:CounterHalfLifeTime 设置半摔周期
* -XX:CompilerThreshold 方法被执行次数阈值