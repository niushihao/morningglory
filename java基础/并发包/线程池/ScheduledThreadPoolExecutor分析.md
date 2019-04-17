## 1、先通过代码回顾下用法
```
// 创建线程池
ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(2);

// 创建可执行任务
Runnable runnable = () -> {
    System.out.println("Thread name ="+Thread.currentThread().getName()+" ,time ="+System.currentTimeMillis());
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
};

// 无延时执行
executor.execute(runnable);

// 5s后触发执行一次
ScheduledFuture<?> begin_scheduled = executor.schedule(runnable, 5, TimeUnit.SECONDS);

// 按固定的频率执行，不受执行时长影响，到点就执行
executor.scheduleAtFixedRate(runnable,0,3,TimeUnit.SECONDS);

//任务执行完后，按固定的延后时间再执行。
executor.scheduleWithFixedDelay(runnable,0,3,TimeUnit.SECONDS);
 ```
 ## 2、代码分析
 #### 2.1 ScheduledThreadPoolExecutor继承了ThreadPoolExecutor，所以要先理解ThreadPoolExecutor。
 #### 2.2 创建线程池
 ```
 public ScheduledThreadPoolExecutor(int corePoolSize) {
        // 调用ThreadPoolExecutor的初始化方法，只是这里指定了延时队列，这也是ScheduledThreadPoolExecutor的重点。
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```
#### 2.3 提交任务
##### 2.3.1 提交方式
```
// 拥有父类的所有提交方式，具体参考ThreadPoolExecutor

// 无延时执行
executor.execute(runnable);

// 5s后触发执行一次
ScheduledFuture<?> begin_scheduled = executor.schedule(runnable, 5, TimeUnit.SECONDS);

// 按固定的频率执行，不受执行时长影响，到点就执行
executor.scheduleAtFixedRate(runnable,0,3,TimeUnit.SECONDS);

//任务执行完后，按固定的延后时间再执行。
executor.scheduleWithFixedDelay(runnable,0,3,TimeUnit.SECONDS);
```
##### 2.3.2 提交任务，这里以scheduleAtFixedRate的方式进行代码分析
```
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,       // 可执行的任务
                                                  long initialDelay,  // 延迟执行的时间，如果为0代表提交后立即执行
                                                  long period         // 执行周期，如果为3 表示每3（unit）执行一次
                                                  ,TimeUnit unit) {   // 时间单位
        // 常规校验                                      
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
            
        // 将任务包装成ScheduledFutureTask类型
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        // 空实现，留给子类实现
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        // 记录当前任务，周期执行时会把他重复放入队列
        sft.outerTask = t;
        // 执行延迟任务
        delayedExecute(t);
        return t;
    }
```
可以看出来，这部分代码就是做了一些校验，然后转化任务格式，最后调用delayedExecute(t)执行任务，下面看下这个方法
```
private void delayedExecute(RunnableScheduledFuture<?> task) {
        // 判断线程池状态，如果不是运行中状态，直接通过拒绝策略处理
        if (isShutdown())
            reject(task);
        else {
            // 先将任务放入队列，这里和ThreadPoolExecutor的实现逻辑不同
            super.getQueue().add(task);
            //再次判断是否可执行
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }
 ```
 这里做了线程池状态的校验，执行逻辑还在ensurePrestart()方法中，继续跟进
 ```
 void ensurePrestart() {
        // 获取当前工作线程数
        int wc = workerCountOf(ctl.get());
        /**
          *如果小于核心线程数，直接调用父类的addWorker
          *addWorker的流程还记得么，是创建一个工作线程(worker)，将任务放入worker，然后启动worker线程
          *worker线程启动后会执行worker中的任务或者从队列取任务
          *
        if (wc < corePoolSize)
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
    }
```
