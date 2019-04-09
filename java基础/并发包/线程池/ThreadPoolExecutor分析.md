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
        这里只介绍submit方式的提交，其他方式差异不大。
   #### 2.2.2 提交任务
   ```
   /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

   ```
   先将callable或者runable类型的任务统一转换成RunableFuture对象，转化后执行核心逻辑execute(ftask),也就是执行任务。
   #### 2.2.3 执行流程
   ```
   public void execute(Runnable command) {
        if (command == null)                                    1.任务不能为空
            throw new NullPointerException();
        int c = ctl.get();                                      2.workerCountOf(c)获取线程池中活跃的线程数
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))                       3.如果当前线程数小于核心线程数，则执行addWorker(command, true)方法
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {         4.如果线程池是运行状态，尝试向队列中放任务
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))        5.双重校验
                reject(command);                                6.异常处理，由初始化线程池时指定的异常处理策略处理改任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);                         7.执行addWorker(null, false)方法
        }
        else if (!addWorker(command, false))                    8.如果上边都不能满足，说明线程池中活跃的线程数以超过核心线程数，而且队列也满了
            reject(command);                                    9.异常处理，由初始化线程池时指定的异常处理策略处理改任务
    }

   ```
可以看到这段代码主要分了三部分判断
 ![image](https://github.com/niushihao/morningglory/blob/master/imange/线程池execute.png)

1. 活跃的线程数小于核心线程数，直接调用addWorker
2. 活跃的线程数大于核心线程数，队列没有满调用workQueue.offer
3. 活跃的线程数大于核心线程数，队列满了调用addWorker
到这我们应该大概了解了每个参数的作用。
#### 2.2.4 创建任务执行者
```
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {                                                                      1.开启死循环
            int c = ctl.get();
            int rs = runStateOf(c);                                                     2.获取当前线程池的运行状态，如果已经shutdown就不在处理新任务

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);                                              3.根据参数core 校验线程数
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))                                  4.cas 将活跃线程数加1
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

boolean workerStarted = false;                                                          5.设置初始标记
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);                                                  6.创建worker
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);                                                 7.将新创建的worker放入数组中，因为有并发问题，所以这里在放之前使用了lock
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;                                             8.更新标记位
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();                                                          9.worker开始执行任务
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
![image](https://github.com/niushihao/morningglory/blob/master/imange/addWorker.png)
#### 2.2.5 执行任务
```
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock();                                                     1.重置初始化worker时的设置，初始化的worker是不能被中断的，因为他就没有开始工作                                              1.
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();                                               2.上锁是为了区分worker是否空闲,非重入锁
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();                                     3.运行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);                     4.
        }
    }
```
![image](https://github.com/niushihao/morningglory/blob/master/imange/runWorker.png)
#### 2.2.6 清理worker
```
private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();                                                        1.加锁，为了下边的集合操作
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();                                                         2.尝试终止空闲worker(空闲的判断条件就是status = 0)

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;            3.timeout的作用范围
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);                                             4.小于核心线程数时 增加worker
        }
    }
 ```
1. worker数量-1<br>
        a. 如果是突然终止，说明是task执行时异常情况导致，即run()方法执行时发生了异常，那么正在工作的worker线程数量需要-1<br>
        b. 如果不是突然终止，说明是worker线程没有task可执行了，不用-1，因为已经在getTask()方法中-1了
2. 从Workers Set中移除worker，删除时需要上锁mainlock<br>
3. tryTerminate()：在对线程池有负效益的操作时，都需要“尝试终止”线程池，大概逻辑：
    判断线程池是否满足终止的状态<br>
        a. 如果状态满足，但还有线程池还有线程，尝试对其发出中断响应，使其能进入退出流程<br>
        b. 没有线程了，更新状态为tidying->terminated
4. 是否需要增加worker线程，如果线程池还没有完全终止，仍需要保持一定数量的线程
    线程池状态是running 或 shutdown<br>
        a. 如果当前线程是突然终止的，addWorker()<br>
        b. 如果当前线程不是突然终止的，但当前线程数量 < 要维护的线程数量，addWorker()
    故如果调用线程池shutdown()，直到workQueue为空前，线程池都会维持corePoolSize个线程，然后再逐渐销毁这corePoolSize个线程


           
