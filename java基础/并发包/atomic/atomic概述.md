## 作用
1. 并发情况下完成无锁模式的原子性操作,通过Unsafe对象完成比较替换或者统计计数的操作。
## 分类
#### 1. 基于基本数据类型包装类的原子类
      AtomicBoolean、AtomicInteger、AtomicLong等等
      他们都是在原子类内部维护了被volatile修饰的对应基本数据类型的变量，通过unsafe进行比较替换或者递增递减操作，但是这几种类型会存在ABA问题。
#### 2. 内置版本号的原子类
      AtomicStampedReference、AtomicMarkableReference
      在比较替换时不仅要预期值的匹配，也会增加预期版本号的匹配，任何一个不能匹配成功都会操作失败，但是AtomicMarkableReference内的版本号是个boolean类型，所以他只能有两种状态，所以这个对象只能减少ABA问题出现的概率，并没有解决ABA问题，AtomicStampedReference解决了ABA问题。
#### 3. 数组原子类
      AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray
      内部维护一个对应类型的数组，每次操作都会根据索引位置获取数组中的值，然后根据这个值去比较替换。
#### 4. 并发计数器
      上边的对象其实都可以用作并发计数器，但是试想一种情况多线程向ConcurrentHashMap中放值的时候，假如向Note添加成功，但是在addCount时通过        incrementAndGet失败了会怎么处理呢，是将刚添加成功的数据删除么，这应该是不太好的。这里他们使用了另一种方式Striped64。
      他的子类有LongAdder、LongAccumulator、DoubleAdder、DoubleAccumulator，Accumulator与Adder的区别是前置支持传入一个计算表达式，而后者只能传入数字，这里以LongAdder进行分析。
```
// 自增
public void increment() {
  add(1L);
}

public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }
  ```  
