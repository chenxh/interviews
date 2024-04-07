
（1）新建状态——（2）就绪状态—（（4）阻塞状态）—（3）运行状态——（5）死亡状态

（1）New：创建线程对象后，该线程处于新建状态，此时它不能运行，和其他Java对象一样，仅仅有Java虚拟机为其分配了内存，没有表现出任何线程的动态特征；
（2）Runnable：线程对象调用了start（）方法后，该线程就进入了就绪状态（也称可运行状态）。处于就绪状态的线程位于可运行池中，此时它只是具备了运行的条件，能否获得CPU的使用权开始运行，还需要等待系统的调度；
（3）Runing：处于就绪状态的线程获得了CPU使用权，开始执行run（）方法中的线程执行体，则线程处于运行状态。当一个线程启动后，它不能一直处于运行状态（除非它的线程执行体足够短，瞬间结束），当使用完系统分配的时间后，系统就会剥脱该线程占用的CPU资源，让其他线程获得执行的机会。只有处于就绪状态的线程才可能转换到运行状态。
（4）Blocked：一个正在执行的线程在某些特殊情况下，如执行耗时的输入/输出操作时，会放弃CPU的使用权，进入阻塞状态。线程进入阻塞状态后，就不能进入排队队列。只有当引用阻塞的原因，被消除后，线程才可以进入就绪状态。
——当线程试图获取某个对象的同步锁时，如果该锁被其他线程所持有，则当前线程进入阻塞状态，如果想从阻塞状态进入就绪状态必须得获取到其他线程所持有的锁。
——当线程调用了一个阻塞式的IO方法时，该线程就会进入阻塞状态，如果想进入就绪状态就必须要等到这个阻塞的IO方法返回。
——当线程调用了某个对象的wait（）方法时，也会使线程进入阻塞状态，notify（）方法唤醒。
——调用了Thread的sleep（long millis）。线程睡眠时间到了会自动进入阻塞状态。
——一个线程调用了另一个线程的join（）方法时，当前线程进入阻塞状态。等新加入的线程运行结束后会结束阻塞状态，进入就绪状态。
线程从阻塞状态只能进入就绪状态，而不能直接进入运行状态，即结束阻塞的线程需要重新进入可运行池中，等待系统的调度。
（5）Terminated：线程的run（）方法正常执行完毕或者线程抛出一个未捕获的异常（Exception）、错误（Error），线程就进入死亡状态。一旦进入死亡状态，线程将不再拥有运行的资格，也不能转换为其他状态。