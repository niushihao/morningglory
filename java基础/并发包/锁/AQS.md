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
                // 尾部节点的前置节点
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
