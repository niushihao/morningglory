
### 目录
* [1、AbstractQueuedSynchronizer](#1abstractqueuedsynchronizer)
    * [1.1、AbstractQueuedSynchronizer拥有的属性](#11abstractqueuedsynchronizer拥有的属性)
    * [1.2、AbstractQueuedSynchronizer获取资源的方法](#12abstractqueuedsynchronizer获取资源的方法)
    * [1.3、借用大佬一个非常好的概括](#13借用大佬一个非常好的概括)
* [2、源码分析](#2源码分析)
    * [2.1、 acquire(int arg)](#21-acquireint-arg)
        * [2.1.1、 tryAcquire(int arg)](#211-tryacquireint-arg)
        * [2.1.2、 addWaiter(Node mode)](#212-addwaiternode-mode)
            * [2.1.2.1、 enq(final Node node)](#2121-enqfinal-node-node)
        * [2.1.3、 acquireQueued(final Node node, int arg)](#213-acquirequeuedfinal-node-node-int-arg)
            * [2.1.3.1、 shouldParkAfterFailedAcquire(p, node)](#2131-shouldparkafterfailedacquirep-node)
            * [2.1.3.2、 parkAndCheckInterrupt()](#2132-parkandcheckinterrupt)
        * [2.1.4、 acquireQueued(final Node node, int arg)总结](#214-acquirequeuedfinal-node-node-int-arg总结)
        * [2.1.5、 acquire(int arg)总结](#215-acquireint-arg总结)
    * [2.2、 release(int arg)](#22-releaseint-arg)
        * [2.2.1、 tryRelease(arg)](#221-tryreleasearg)
        * [2.2.2、 unparkSuccessor(h)](#222-unparksuccessorh)
        * [2.2.3、 release(int arg)总结](#223-releaseint-arg总结)
    * [2.3、 acquireShared(int arg)](#21-acquireint-arg)
        * [2.1.1、 doAcquireShared(arg)](#211-tryacquireint-arg)
        * [2.1.2、 setHeadAndPropagate(node, r)](#212-addwaiternode-mode)
    * [2.4、 releaseShared(int arg)](#21-acquireint-arg)

### 1、AbstractQueuedSynchronizer
它是一个抽象队列同步器，这个对象我还没有直接用到过，但是它却是juc的基础，因为他是Lock,CountDownLatch的底层实现，所以还是要了解这个对象的功能。
#### 1.1、AbstractQueuedSynchronizer拥有的属性
```
// 由两个node组成链表(网上都叫队列)，当出现并发冲突时，竞争失败的线程会放入队列
private transient volatile Node head;
private transient volatile Node tail;

// 代表共享资源，当一个线程获取资源后会改变他的状态
private volatile int state;

// state对应的方法
getState()
setState(int newState)
compareAndSetState(int expect, int update)
```
可以看到AQS内部维护了一个队列(链表)，和一个状态值，而且这些属性都是用volatile关键字修饰，下面先看下Node的结构。
```
static final class Node {
        // 这两个属性说明Node是有共享/独占两种类型
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;

        /**
          * 初始值 0
          * CANCELLED =  1 代表已取消
          * SIGNAL = -1 代表等待被唤醒
          * CONDITION = -2 和condition配合使用
          * PROPAGATE = -3 代表无条件传播
        volatile int waitStatus;
        
        // 将队列数据连接起来
        volatile Node prev;
        volatile Node next;

        // 保存当前等待的线程
        volatile Thread thread;
        。。。。
 }
 ```
 #### 1.2、AbstractQueuedSynchronizer获取资源的方法
 1. tryAcquire(int arg)         :独占方式尝试获取资源
 2. tryRelease(int arg)         :独占方式释放资源<br>
 3. tryAcquireShared            :共享方式获取资源
 4. tryReleaseShared            :共享方式释放资源
 5. isHeldExclusively()         :该线程是否正在独占资源。只有用到condition才需要去实现它。
 #### 1.3、借用大佬一个非常好的概括
以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。
### 2、源码分析
依照acquire-release、acquireShared-releaseShared的次序来。
#### 2.1 acquire(int arg)
```
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
函数执行流程
1. tryAcquire(arg)尝试获取资源，如果成功直接返回
2. 将任务标记为独占方式，并放入队列尾部
3. 自旋获取资源，返回当前线程释放被中断标记
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。
##### 2.1.1 tryAcquire(int arg)
```
// 具体是由子类实现的
protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }
```
##### 2.1.2 addWaiter(Node mode)
```
private Node addWaiter(Node mode) {
        // 创建一个给定模式(独占/共享)的节点
        Node node = new Node(Thread.currentThread(), mode);
        // 尝试直接在放在尾部节点，如果成功则返回
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // Node还没有初始化或者cas失败执行
        enq(node);
        return node;
    }
 ```
 ###### 2.1.2.1 enq(final Node node)
 ```
 private Node enq(final Node node) {
        // 开启死循环
        for (;;) {
            Node t = tail;
            // 如果尾部节点为空(还没有初始化)，则创建头部和尾部的节点
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                // 继续尝试将当前节点放入尾部节点，直到成功
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
##### 2.1.3 acquireQueued(final Node node, int arg)
```
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            // 当前线程中断标记
            boolean interrupted = false;
            for (;;) {
                // 节点的前置节点
                final Node p = node.predecessor();
                // 如果前置节点为head节点，并且尝试获取资源成功(只有head节点线程释放锁才会成功)
                if (p == head && tryAcquire(arg)) {
                    // 将当前节点设置为head节点
                    setHead(node);
                    // 将head节点的后置节点至为null(从当前队列移除)
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 判断当前线程是否应该被阻塞
                if (shouldParkAfterFailedAcquire(p, node) &&
                    // 将当前线程阻塞并判断线程是否已经被中断
                    parkAndCheckInterrupt())
                    // 设置线程中断标记
                    interrupted = true;
            }
        } finally {
            // 异常终止时，取消当前node节点竞争资源
            if (failed)
                cancelAcquire(node);
        }
    }
 ```
###### 2.1.3.1 shouldParkAfterFailedAcquire(p, node)
```
// 方法如果返回false,将会一直循环调用
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        /**
          * 前置节点的状态
          * CANCELLED =  1 代表已取消
          * SIGNAL = -1 代表等待被唤醒
          * CONDITION = -2 和condition配合使用
          * PROPAGATE = -3 代表无条件传播
        int ws = pred.waitStatus;
        // 等待被唤醒
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        // 节点已被取消
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             * 一直向前寻找，知道找到一个没有被取消的节点(这中间被取消的节点会从队列中移除)
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             * 将前置节点的状态至为SIGNAL
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
整个流程其实就是要找到一个没有被取消的前置节点，并且将前置节点的状态设置为SIGNAL，用来告诉前置节点获取到资源后通知自己，然后自己就可以安心休息了
，该方法返回false会一直重复执行，返回true,会执行parkAndCheckInterrupt()
###### 2.1.3.2 parkAndCheckInterrupt()
```
private final boolean parkAndCheckInterrupt() {
        // 阻塞当前线程
        LockSupport.park(this);
        // 返回当前线程是否被中断，并重置中断状态
        return Thread.interrupted();
    }
```
##### 2.1.4 acquireQueued(final Node node, int arg)总结
1. 尝试将当前阶段放入尾部节点，如果成功直接返回
2. 寻找没有被取消的前置节点(同时清理已取消的前置节点)，设置其状态为SIGNAL，用来获取资源后通知自己
3. 阻塞当前线程，返回线程释放被中断标记
4. 重试以上步骤 知道步骤1成功
5. 如果方法异常终止，则将当前节点从队列移除。
4. 重试以上步骤 知道步骤1成功方法
4. 重试以上步骤 知道步骤1成功
##### 2.1.5 acquire(int arg)总结
1. 尝试获取资源，由子类实现
2. 自旋直到线程成功获取资源，并返回当前线程是否被中断。如果没有被中断，则获取资源成功，否则进行自我中断。
![image](https://github.com/niushihao/morningglory/blob/master/imange/acquire.png)
这也就是ReentrantLock.lock()的流程，不信你去看其lock()源码吧，整个函数就是一条acquire(1)！！！

#### 2.2 release(int arg)
```
public final boolean release(int arg) {
        // 尝试释放资源
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                // 唤醒后置节点的线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
##### 2.2.1 tryRelease(arg)
```
protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
```
和之前套路一致，也是由子类实现
##### 2.2.2 unparkSuccessor(h)
```
private void unparkSuccessor(Node node) {
        /*
         * 改变节点的状态
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
            
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            // 如果头节点的后置节点为null，则倒排查询出最前边的没有被取消的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // 唤醒后置节点中被阻塞的线程
        if (s != null)
            LockSupport.unpark(s.thread);
    }
 ```
##### 2.2.3 release(int arg)总结
1. 尝试释放资源，失败直接返回
2. 唤醒head节点后第一个需要被唤醒(waitStatus < 0)的节点<br>

这里在回想下acquireQueued()方法，release方法唤醒了后置节点的线程，然后它继续判断，此时他的前置节点为head节点，如果尝试获取资源成功，说明release中的tryRelease（将state变为0）已经成功，所以讲自己设为head节点，然后这个方法返回，此线程就结束了在AQS中的等待旅程，开始做自己的事情了。
```
// 节点的前置节点
final Node p = node.predecessor();
// 如果前置节点为head节点，并且尝试获取资源成功(只有head节点线程释放锁才会成功)
if (p == head && tryAcquire(arg)) {
    // 将当前节点设置为head节点
    setHead(node);
    // 将head节点的后置节点至为null(从当前队列移除)
    p.next = null; // help GC
    failed = false;
    return interrupted;
}
```
#### 2.3 acquireShared(int arg)
```
public final void acquireShared(int arg) {
        // 尝试获取资源
        if (tryAcquireShared(arg) < 0)
            // 获取失败后进入队列
            doAcquireShared(arg);
    }
```
此处看不到太多细节，但是独占模式的tryAcquire是返回boolean,而共享模式的tryAcquireShared是返回一个数字，这有什么区别呢，我们继续进入doAcquireShared找答案。
##### 2.3.1 doAcquireShared(arg)
```
private void doAcquireShared(int arg) {
        // 以共享模式加入队列
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 获取前驱节点
                final Node p = node.predecessor();
                if (p == head) {
                    // 再次尝试获取资源
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        // 设置新的头结点，并根据r释放资源
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                // 与独占模式相同
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
这个方法大体上和独占处理方式是一样的，不同点就是setHeadAndPropagate(node, r)方法，而且这个方法用到了tryAcquireShared的返回值，所以为了搞清楚这个返回值和tryAcquire返回的boolean的区别，还得往下看。
###### 2.3.1.1 setHeadAndPropagate(node, r)
```
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        // 将当前节点设置为头节点
        setHead(node);
        // 只有tryAcquireShared返回值 > 0或者waitStatus < 0才继续
        // 这也是共享模式比独占模式多的逻辑
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            // 只有后置节点是共享类型，就唤醒
            if (s == null || s.isShared())
                // 唤醒节点
                doReleaseShared();
        }
    }
 // releaseShared(int arg)用的就是这个方法   
 private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }

```
##### 2.3.2 doAcquireShared(arg)总结
1. 以共享模式放入等待队列
2. 如果前置节点是头结点，再次尝试获取资源
3. 获取成功后，将当前节点设置为头结点，并且根据tryAcquireShared返回值 > 0决定是否唤醒后记节点(但是这个判断都是用 ||，所以会出现即使==0也会唤醒一些不必要的节点)
4. 获取失败，和独占模式处理相同。
所以综合看，就比独占模式多了一步释放后续节点的操作(虽然判断条件有点宽)，也是就只要tryAcquireShared返回值 > 0，队列中的多个节点是可以并发运行的。毕竟是获取共享资源。
#### 2.4 releaseShared(int arg)
此方法在acquireShared(int arg)中已经说过，不在重复。
