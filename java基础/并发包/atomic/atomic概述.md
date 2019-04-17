## 作用
1. 并发情况下完成无锁模式的原子性操作,通过Unsafe对象完成比较替换或者统计计数的操作。
## 分类
#### 1. 基于基本数据类型包装类的原子类
      AtomicBoolean、AtomicInteger、AtomicLong等等
      他们都是在原子类内部维护了被volatile修饰的对应基本数据类型的变量，通过unsafe进行比较替换或者递增递减操作，但是这几种类型会存在ABA问题。
#### 2. 内置版本号的原子类
      AtomicStampedReference、AtomicMarkableReference
      在比较替换时不仅要预期值的匹配，也会增加预期版本号的匹配，任何一个不能匹配成功都会操作失败，</br>但是AtomicMarkableReference内的版本号是个boolean类型，</br>所以他只能有两种状态，所以这个对象只能减少ABA问题出现的概率，并没有解决ABA问题，AtomicStampedReference解决了ABA问题。
#### 3. 数组原子类
      AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray
      内部维护一个对应类型的数组，每次操作都会根据索引位置获取数组中的值，然后根据这个值去比较替换。
#### 4. 并发计数器
      上边的对象其实都可以用作并发计数器，但是试想一种情况多线程向ConcurrentHashMap中放值的时候，假如向Note添加成功，但是在addCount时通过        incrementAndGet失败了会怎么处理呢，是将刚添加成功的数据删除么，这应该是不太好的。这里他们使用了另一种方式Striped64。
      他的子类有LongAdder、LongAccumulator、DoubleAdder、DoubleAccumulator，Accumulator与Adder的区别是前置支持传入一个计算表达式，而后者只能传入数字，LongAdder没有自己的属性都是使用的父类的，所有这里只看Striped64。
```
 /** Number of CPUS, to place bound on table size */
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    /**
     * Table of cells. When non-null, size is a power of 2.
     */
    transient volatile Cell[] cells;

    /**
     * Base value, used mainly when there is no contention, but also as
     * a fallback during table initialization races. Updated via CAS.
     */
    transient volatile long base;

    /**
     * Spinlock (locked via CAS) used when resizing and/or creating Cells.
     */
    transient volatile int cellsBusy;
```
实现原理就是当需要增加计数时，先尝试通过cas向base设置，如果成功直接返回，如果失败就初始化Cell数组，通过取模获取数组的索引，如果当前位置没有值就创建一个Cell并放入数组，如果有值就更新改值，最后统计结果的时候      
```
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        if ((h = getProbe()) == 0) { //获取当前线程的probe值，如果为0，则需要初始化该线程的probe值
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            if ((as = cells) != null && (n = as.length) > 0) { //获取cell数组
                if ((a = as[(n - 1) & h]) == null) { // 通过（hashCode & (length - 1)）这种算法来实现取模
                    if (cellsBusy == 0) {       // 如果当前位置为null说明需要初始化
                        Cell r = new Cell(x);   // Optimistically create
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                } 
                //运行到此说明cell的对应位置上已经有想相应的Cell了，不需要初始化了
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                    
                //尝试去修改a上的计数，a为Cell数组中index位置上的cell
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                    
                //cell数组最大为cpu的数量，cells != as表面cells数组已经被更新了    
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1]; //Cell数组扩容，每次扩容为原来的两倍
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }

  ```  
