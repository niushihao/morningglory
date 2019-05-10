这篇会用到AQS的知识。
### 三种锁类型
1. ReentrantLock 重入锁
    1.1 公平重入锁
    1.2 非公平重入锁
2. ReentrantReadWriteLock 读写锁
    2.1 读锁
        2.1.1 公平读锁
        2.1.2 非公平读锁
    2.2 写锁
        2.2.1 公平写锁
        2.2.2 非公平写锁
3. StampedLock
    3.1 乐观读锁
    3.2 读锁
    3.3 写锁
