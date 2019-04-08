## 1、首先通过一段代码回顾下用法
```
// a.创建线程池
ThreadPoolExecutor executor = new ThreadPoolExecutor(3,6,0L
        , TimeUnit.SECONDS,new ArrayBlockingQueue<Runnable>(100));
// b.提交任务
Future<Integer> future = executor.submit(() -> {
    return 1+2;
});
// c.获取结果
Integer integer = future.get();
```
可以看到一共分为三不操作
        a.创建线程池
        b.提交可执行任务
        c.获取任务执行结果
## 2、代码分析
### 2.1 创建线程池
```
/**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
    public ThreadPoolExecutor(int corePoolSize,                         *核心线程数*
                              int maximumPoolSize,                      最大线程数
                              long keepAliveTime,                       空闲存活时间（默认不控制核心线程的存活时间，也可以通过allowCoreThreadTimeOut 设置核心线程的存活时间）
                              TimeUnit unit,                            时间单位
                              BlockingQueue<Runnable> workQueue,        存放任务的阻塞队列
                              ThreadFactory threadFactory,              创建线程的工厂
                              RejectedExecutionHandler handler) {       异常处理策略
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
   ```
   这里边是没什么逻辑的，就是初始化对象中的参数，和一些校验，对于每个属性的用法在 任务提交的时候再看。
   ### 2.2 提交任务
   #### 2.2.1 提交方式
        1.void execute(Runnable command)                                                处理无返回值的任务
        2.<T> Future<T> submit(Callable<T> task)                                        处理有返回值的任务返回Future对象
        3.<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)        批量处理有返回值的任务，返回任务执行后的结果，其实就是在内部帮我们循环调用了future.get()方法。
