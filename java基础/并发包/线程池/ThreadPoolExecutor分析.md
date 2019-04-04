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
