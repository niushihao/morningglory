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
            
        // 将任务包装成ScheduledFutureTask类型，为什么要包装成这个对象呢，这个下边说
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
          *worker线程启动后会执行worker中的任务或者从队列取任务去执行
          *这里设置一个null,说明worker启动后只能从队列去任务
          *这里没有对wc > = corePoolSize做处理，是应为在上边方法中已经将任务先放入队列了
        if (wc < corePoolSize)
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
    }
```
如果只是看到这，我们没有发现任何和延迟处理有关的逻辑，那他是怎么做到的呢？还记得上边有个将我们提交的任务转化为ScheduledFutureTask类型的操作么，
没错，这个对象不仅记录了一些必要参数外，还重写了run()方法，run()方法在addWorker后由父类调用，继续看下run()的代码
```
public void run() {
            /**
               *判断是否需要周期执行，判断条件就是这个属性 != 0,
               *而调用executor.scheduleAtFixedRate(runnable,0,3,TimeUnit.SECONDS)的时候我们给他设置了3
            */
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            // 如果不是周期任务，直接执行
            else if (!periodic)
                ScheduledFutureTask.super.run();
            else if (ScheduledFutureTask.super.runAndReset()) {
                setNextRunTime();
                reExecutePeriodic(outerTask);
            }
        }
```
这里逻辑比较清晰，先判断当前任务是否为周期任务，如果不是，直接执行，如果是也是先执行并重置状态，但是多了下边两个方法
```
// 比较重要的方法，获取任务下次执行的时间
private void setNextRunTime() {
            long p = period;
            /**
              *如果大于0 ，下次执行时间就是上次执行时间+间隔周期
              *而我们传的是个3 是大于0 的，所以我们可以确认scheduleAtFixedRate方法是按照固定周期执行的
              *而不关心任务执行过程消耗的时间
            if (p > 0)
                time += p;
            /**
              *什么时候是小于0呢
              *其实这就是scheduleWithFixedDelay 与 scheduleAtFixedRate唯一的不同点
              *此时的下次执行时间是 当前时间+周期间隔，所以我们可以确认其实这就是scheduleWithFixedDelay是
              *每次任务结束后 等待p(unit)在执行，也就是会受任务执行时间的影响
            else
                time = triggerTime(-p);
        }
```
reExecutePeriodic方法名字应该能猜到，将任务下次执行时间计算好后有重新放入队列等待执行。
```
void reExecutePeriodic(RunnableScheduledFuture<?> task) {
        if (canRunInCurrentRunState(true)) {
            super.getQueue().add(task);
            if (!canRunInCurrentRunState(true) && remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }
```
分析到这我们总结下流程：
1. 创建线程池
2. 提交任务后将任务变成ScheduledFutureTask类型，变调用父类的方法执行任务
3. 重写执行逻辑，计算周期任务的下次执行时间，并重新放入队列等待执行
其实到这逻辑大概清楚了，但是有一个非常重要的点就是放入的任务，如何保证按照可执行时间的先后顺序执行的呢，还记得初始化时默认指定了一个延迟队列么，其实就是在那里做了排序。下边看下DelayedWorkQueue
## DelayedWorkQueue优先级队列
这里就不贴源码了，原理就是重写了add,take的方法，在放入或者取出的时候会根据优先级(任务执行时间排序)按照堆排序思想，将顺序维护好，然后取值的时候判断任务执行时间是否>=当前时间，如果是就取出执行，否则等待到达时间才取出。另外还有一点队列中存放的是任务RunnableScheduledFuture，而这个对象间接继承了Compare接口，自定义了按执行时间排序的规则，所以放入队列的时候可以通过compareto去比较。
关于堆排序详细介绍，后边单独补充。
