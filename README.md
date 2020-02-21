# completionservice-demo
completion service demo


# ==========  异步并发利器：实际项目中使用CompletionService提升系统性能的一次实践


git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/13539956066/Completion-CyclicBarrier.git
git push -u origin master


Completion-CyclicBarrier
https://www.cnblogs.com/javazhiyin/p/12336838.html


实现
业务
投保业务主要涉及这几个大的方面：投保校验、核保校验、保费试算

投保校验：最主要的是要查询客户黑名单和风险等级，都是千万级的表。而且投保人和被保人都需要校验

核保校验：除了常规的核保规则校验，查询千万级的大表，还需要调用外部智能核保接口获得用户的风险等级，投保人和被保人都需要校验

保费试算：需要计算每个险种的保费

设计
根据上面的业务，如果串行执行的话，单次性能肯定不高，所以考虑多线程异步执行获得校验结果，再对结果综合判断

投保校验：采用一个线程(也可以根据投保人和被保人数量来采用几个线程)

核保校验：

常规校验：采用一个线程

外部调用：有几个用户(指投保人和被保人)就采用几个线程

保费计算：有几个险种就采用几个线程，最后合并得到整个的保费


# ================== 【Java并发工具类】CountDownLatch和CyclicBarrier


https://www.cnblogs.com/myworld7/p/12337246.html


【Java并发工具类】CountDownLatch和CyclicBarrier
阅读目录
前言
CountDownLatch和CyclicBarrier的用途介绍
CountDownLatch
CyclicBarrier
在对账系统中使用CountDownLatch和CyclicBarrier
利用并行优化对账系统
使用CountDownLatch实现线程等待
使用CyclicBarrier进一步优化对账系统
小结
 

回到目录
前言
下面介绍协调让多线程步调一致的两个工具类：CountDownLatch和CyclicBarrier。

回到目录
CountDownLatch和CyclicBarrier的用途介绍
CountDownLatch
// API
 void       await(); // 使当前线程在闭锁计数器到零之前一直等待，除非线程被中断。
 boolean    await(long timeout, TimeUnit unit); // 使当前线程在闭锁计数器至零之前一直等待，除非线程被中断或超出了指定的等待时间。
 void       countDown(); // 递减闭锁计数器，如果计数到达零，则释放所有等待的线程。
 long       getCount(); // 返回当前计数。
 String     toString(); // 返回标识此闭锁及其状态的字符串。
CountDownLatch是一个同步工具类，在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待。可以指定计数初始化CountDownLatch，当调用countDown()方法后，在当前计数到达零之前，await()方法会一直受阻塞。计数到达零之后，所有被阻塞的线程都会被释放，await()的所有后续调用都会立即返回。CountDownLatch的计数只能被使用一次，如果需要重复计数使用，则要考虑使用CyclicBarrier。

CountDownLatch的用途有很多。将计数为1初始化的CountDownLatch可用作一个简单的开/关或入口：在通过调用countDown()的线程打开入口前，所有调用await()的线程都一直在入口出等待。而用N初始化CountDownLatch可以使一个线程在N个线程完成某项操作之前一直等待，或者使其在某项操作完成N次之前一直等待。

COuntDownLatch的内存一致性语义：线程中调用 countDown() 之前的操作 Happens-Before紧跟在从另一个线程中对应 await() 成功返回的操作。

CyclicBarrier
// API
 int        await(); // 线程将一直等待直到所有参与者都在此 barrier 上调用 await 方法
 int        await(long timeout, TimeUnit unit); // 线程将一直等待直到所有参与者都在此 barrier 上调用 await 方法, 或者超出了指定的等待时间。
 int        getNumberWaiting(); // 返回当前在屏障处等待的参与者数目。
 int        getParties(); // 返回要求启动此 barrier 的参与者数目。
 boolean    isBroken(); // 查询此屏障是否处于损坏状态。
 void       reset(); // 将屏障重置为其初始状态。
CyclicBarrier是一个同步辅助类，它允许一组线程互相等待，直到到达某个公共屏障点（barrier也可被翻译为栅栏） (common barrier point)。 CyclicBarrier 适用于在涉及一组固定大小的线程的程序中，这些线程必须不时地互相等待的情况。即所有线程都必须到达屏障位置后，下面的程序才能继续执行，适于在迭代算法中使用。因为 barrier 在释放等待线程后可以计数器会被重置可继续使用，所以称它为循环 的 barrier。

CyclicBarrier支持一个可选的 Runnable命令（也就是可以传入一个线程执行其他操作），在一组线程中的最后一个线程到达之后（但在释放所有线程之前），该命令将只在每个 barrier point 运行一次。这对所有参与线程继续运行之前更新它们的共享状态将十分有用。

CyclicBarrier的内存一致性语义：线程中调用 await() 之前的操作 Happens-Before 那些是屏障操作的一部份的操作，后者依次 Happens-Before 紧跟在从另一个线程中对应 await() 成功返回的操作。

Actions in a thread prior to calling await() happen-before actions that are part of the barrier action, which in turn happen-before actions following a successful return from the corresponding await() in other threads.
回到目录
在对账系统中使用CountDownLatch和CyclicBarrier
对账系统流程图如下：

image-20200220174328582

目前对账系统的处理流程是：先查询订单，然后查询派送单，之后对比订单和派送单，将差异写入差异库。对账系统的代码抽象后如下：

while(存在未对账订单){
    // 查询未对账订单
    pos = getPOrders();
    // 查询派送单
    dos = getDOrders();
    // 执行对账操作
    diff = check(pos, dos);
    // 差异写入差异库
    save(diff);
}
利用并行优化对账系统
目前的对账系统，由于订单量和派送单量巨大，所以查询未对账订单getPOrder()和查询派送单getDOrder()都相对比较慢。目前对账系统是单线程执行的，示意图如下（图来自参考[1]）：

image-20200220183528662

对于串行化的系统，优化性能首先想到的就是能否利用多线程并行处理。
如果我们能将getPOrders()和getDOrders()这两个操作并行处理，那么将会提升效率很多。因为这两个操作并没有先后顺序的依赖，所以，我们可以并行处理这两个耗时的操作。
并行后的示意图如下（图来自参考[1]）：

image-20200220183728865

对比单线程的执行示意图，我们发现在同等时间里，并行执行的吞吐量近乎单线程的2倍，优化效果还是相对明显的。

优化后的代码如下：

while(存在未对账订单){
    // 查询未对账订单
    Thread T1 = new Thread(()->{
        pos = getPOrders();
    });
    T1.start();

    // 查询派送单
    Thread T2 = new Thread(()->{
        dos = getDOrders();
    });
    T2.start();

    // 要等待线程T1和T2执行完才能执行check()和save()这两个操作
    // 通过调用T1.join()和T2.join()来实现等待
    // 当T2和T2线程退出时，调用T1.jion()和T2.join()的主线程就会从阻塞态被唤醒，从而执行check()和save()
    T1.join();
    T2.join();

    // 执行对账操作
    diff = check(pos, dos);
    // 差异写入差异库
    save(diff);
}
使用CountDownLatch实现线程等待
上面的解决方案美中不足的地方在于：每一次while循环都会创建新的线程，而线程的创建是一个耗时操作。所以，最好能使创建出来的线程能够循环使用。一个自然而然的方案便是线程池。

// 创建 2 个线程的线程池
Executor executor =Executors.newFixedThreadPool(2);
while(存在未对账订单){
    // 查询未对账订单
    executor.execute(()-> {
        pos = getPOrders();
    });

    // 查询派送单
    executor.execute(()-> {
        dos = getDOrders();
    });

    /* ？？如何实现等待？？*/

    // 执行对账操作
    diff = check(pos, dos);
    // 差异写入差异库
    save(diff);
}   
于是我们就创建两个固定大小为2的线程池，之后在while循环里重复利用。
但是问题也出来了：主线程如何得知getPOrders()和getDOrders()这两个操作什么时候执完？
前面主线程通过调用线程T1和T2的join()方法来等待T1和T2退出，但是在线程池的方案里，线程根本就不会退出，所以，join()方法不可取。

这时我们就可以使用CountDownLatch工具类，将其初始计数值设置为2。当执行完pos = getPOrders();后，将计数器减一，执行完dos = getDOrders();后也将计数器减一。当计数器为0时，被阻塞的主线程就可以继续执行了。

// 创建 2 个线程的线程池
Executor executor = Executors.newFixedThreadPool(2);

while(存在未对账订单){
    // 计数器初始化为 2
    CountDownLatch latch = new CountDownLatch(2);
    // 查询未对账订单
    executor.execute(()-> {
        pos = getPOrders();
        latch.countDown();    // 实现对计数器减1
    });

    // 查询派送单
    executor.execute(()-> {
        dos = getDOrders();
        latch.countDown();    // 实现对计数器减1
    });

    // 等待两个查询操作结束
    latch.await(); // 在await()返回之前，主线程会一直被阻塞

    // 执行对账操作
    diff = check(pos, dos);
    // 差异写入差异库
    save(diff);
}
使用CyclicBarrier进一步优化对账系统
除了getPOrders()和getDOrders()这两个操作可以并行，这两个查询操作和check() 、save()这两个对账操作之间也可以并行。

image-20200220185636511

两次查询操作和对账操作并行，对账操作还依赖查询操作的结果，有点像生产者-消费者的意思，两次查询操作是生产者，对账操作是消费者。那么，我们就需要一个队列，来保存生产者生产的数据，而消费者则从这个队列消费数据。

不过，针对对账系统，可以设计两个队列，并且这两个队列之间还有对应关系。订单查询操作将订单查询结果插入订单队列，派送单查询操作将派送单插入派送单队列，这两个队列的元素之间是有一一对应关系。这样的好处在于：对账操作可以每次从订单队列出一个元素和从派送单队列出一个元素，然后对这两个元素执行对账操作，这样数据一定不会乱掉。

image-20200220190145881

如何使两个队列实现完全的并行？
两个查询操作所需时间并不相同，那么一个简单的想法便是，一个线程T1执行订单的查询工程，一个线程T2执行派送单的查询工作，仅当线程T1和T2各自都生产完1条数据的时候，通知线程T3执行对账操作。

image-20200220190749565

先查询完的一方需要在设置的屏障点等待另一方，直到双方都到达屏障点，才开始继续下一步任务。

于是我们可以使用CyclicBarrier来实现这个功能。创建一个计数器初始值为2的CyclicBarrier，同时传入一个回调函数，当计数器减为0的时候，便调用这个函数。

Vector<P> pos; // 订单队列
Vector<D> dos; // 派送单队列
// 执行回调的线程池 
// 固定线程数量为1是因为只有单线程取获取两个队列中的数据才不会出现数据匹配不一致问题
Executor executor = Executors.newFixedThreadPool(1); 
// 创建CyclicBarrier的计数器为2，传入一个线程另外执行对账操作
// 当计数器为0时,会运行传入线程执行对账操作
final CyclicBarrier barrier = new CyclicBarrier(2, ()->{
                                    executor.execute(()->check());
                             });
void check(){
    P p = pos.remove(0); // 从订单队列中获取订单
    D d = dos.remove(0); // 从派送单队列中获取派送单
    // 执行对账操作
    diff = check(p, d);
    // 差异写入差异库
    save(diff);
}

void checkAll(){
    // 循环查询订单库
    Thread T1 = new Thread(()->{
        while(存在未对账订单){
            pos.add(getPOrders()); // 查询订单库
            barrier.await(); // 将计数器减一并等待直到计数器为0
        }
    });
    T1.start();  
    // 循环查询运单库
    Thread T2 = new Thread(()->{
        while(存在未对账订单){
            dos.add(getDOrders()); // 查询运单库
            barrier.await(); // 将计数器减一并等待直到计数器为0
        }
    });
    T2.start();
}
线程T1负责查询订单，当查出一条时，调用barrier.await()来将计数器减1，同时等待计数器变为0；线程T2负责查询派送订单，当查出一条时，也调用barrier.await()来将计数器减1，同时等待计数器变为0；当T1和T2都调用barrier.await()时，计数器就会减到0，此时T1和T2就可以执行下一条语句了，同时会调用barrier的回调函数来执行对账操作。

CyclicBarrier的计数器有自动重置的功能，当减到0时，会自动重置你设置的初始值。于是，我们便可以重复使用CyclicBarrier。

回到目录
小结
CountDownLatch和CyclicBarrier是Java并发包提供的两个非常易用的线程同步工具类。它们的区别在于：CountDownLatch主要用来解决一个线程等待多个线程的场景（计数器一旦减到0，再有线程调用await()，该线程会直接通过，计数器不会被重置）；CyclicBarrier是一组线程之间的相互等待（计数器可以重用，减到0会重置为设置的初始值），还可以传入回调函数，当计数器为0时，执行回调函数。




