## jvm
JVM 是可运行 Java 代码的假想计算机 ，包括一套字节码指令集、一组寄存器、一个栈、一个垃圾回收，堆 和 一个存储方法域。JVM 是运行在操作系统之上的，它与硬件没有直接的交互。

## jvm 运行过程
Java 源文件—->编译器—->字节码文件
字节码文件—->JVM—->机器码


## jvm 中的线程
JVM 允许一个应用并发执行多个线程。
Hotspot JVM 中的 Java 线程与原生操作系统线程有直接的映射关系。当线程本地存储、缓
冲区分配、同步对象、栈、程序计数器等准备好以后，就会创建一个操作系统原生线程。
Java 线程结束，原生线程随之被回收。操作系统负责调度所有线程，并把它们分配到任何可用的 CPU 上。
当原生线程初始化完毕，就会调用 Java 线程的 run() 方法。当线程结束时，会释放原生线程和 Java 线程的所有资源。

## Hotspot JVM 后台运行的系统线程
* 虚拟机线程（VM thread）：这个线程等待 JVM 到达安全点操作出现。这些操作必须要在独立的线程里执行，因为当堆修改无法进行时，线程都需要 JVM 位于安全点。这些操作的类型有：stop-the-world 垃圾回收、线程栈 dump、线程暂停、线程偏向锁（biased locking）解除
* 周期性任务线程：这线程负责定时器事件（也就是中断），用来调度周期性操作的执行
* GC 线程：这些线程支持 JVM 中不同的垃圾回收活动。
* 编译器线程：这些线程在运行时将字节码动态编译成本地平台相关的机器码。
* 信号分发线程：这个线程接收发送到 JVM 的信号并调用适当的 JVM 方法处理。

## JVM 内存区域
内存区域包含以下区域
* 线程私有
  * 程序计数器 PC: 指向虚拟机字节码指令的位置。 无OOM 异常
  * 虚拟机栈
    * 和线程的生命周期相同
    * 每调用一次方法就push一个栈帧
    * 栈帧的结构：本地变量表，操作数栈，对运行时常量池的引用
    * 异常：stackoverflowerror， outofmemoryerror
  * 本地方法栈
* 线程共享
  * 方法区（method area）：运行时常量池
  * 类实例区（java 堆）
    * 新生代：eden，from survivor， to survivor
    * 老年代
    * 异常：outofmemoryerror
* 直接内存：不受JVM管理

**MinorGC 的过程**
MinorGC 采用复制算法， 过程如下：
1：eden、servicorFrom 复制到 ServicorTo，年龄+1
首先，把 Eden 和 ServivorFrom 区域中存活的对象复制到 ServicorTo 区域（如果有对象的年龄以及达到了老年的标准，则赋值到老年代区），同时把这些对象的年龄+1（如果 ServicorTo 不够位置了就放到老年区）

2：清空 eden、servicorFrom
然后，清空 Eden 和 ServicorFrom 中的对象；

3：ServicorTo 和 ServicorFrom 互换
最后，ServicorTo 和 ServicorFrom 互换，原 ServicorTo 成为下一次 GC 时的 ServicorFrom
区。

**JAVA8 与元数据**
在 Java8 中，永久代已经被移除，被一个称为“元数据区”（元空间）的区域所取代。元空间的本质和永久代类似，元空间与永久代之间最大的区别在于：***元空间并不在虚拟机中，而是使用本地内存***。因此，默认情况下，元空间的大小仅受本地内存限制。类的元数据放入 native memory, 字符串池和类的静态变量放入 java 堆中，这样可以加载多少类的元数据就不再由MaxPermSize 控制, 而由系统的实际可用空间来控制。



