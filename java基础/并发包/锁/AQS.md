## AbstractQueuedSynchronizer源码分析
它是一个抽象队列同步器，这个对象我还没有直接用到过，但是它却是juc的基础，因为他是Lock,CountDownLatch的底层实现，所以还是要了解这个对象的功能。
### AbstractQueuedSynchronizer拥有的属性
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
 上边只是说了AQS中的属性值，可能此时还不能理解他到底是怎么工作的，为了方便，我们还是结合一个具体的是用场景进行分析。
 


/**
