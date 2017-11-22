---
layout: post
published: true
title: ConcurrentHashMap --Size相关的方法
---
# ConcurrentHashMap 总结（ 下 ）

## 2.8 Size相关的方法

对于ConcurrentHashMap来说，这个table里到底装了多少东西其实是个不确定的数量，因为不可能在调用size()方法的时候像GC的“stop the world”一样让其他线程都停下来让你去统计，因此只能说这个数量是个估计值。对于这个估计值，ConcurrentHashMap也是大费周章才计算出来的。

### 2.8.1 辅助定义

为了统计元素个数，ConcurrentHashMap定义了一些变量和一个内部类

	/**
     * A padded cell for distributing counts.  Adapted from LongAdder
     * and Striped64.  See their internal docs for explanation.
     */
    @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }
 
  /******************************************/ 
 
    /**
     * 实际上保存的是hashmap中的元素个数  利用CAS锁进行更新
     但它并不用返回当前hashmap的元素个数 
 
     */
    private transient volatile long baseCount;
    /**
     * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
     */
    private transient volatile int cellsBusy;
 
    /**
     * Table of counter cells. When non-null, size is a power of 2.
     */
    private transient volatile CounterCell[] counterCells;

### 2.8.2 mappingCount与Size方法

mappingCount与size方法的类似  从Java工程师给出的注释来看，应该使用mappingCount代替size方法 两个方法都没有直接返回basecount 而是统计一次这个值，而这个值其实也是一个大概的数值，因此可能在统计的时候有其他线程正在执行插入或删除操作。

    public int size() {
            long n = sumCount();
            return ((n < 0L) ? 0 :
                    (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                    (int)n);
        }
         /**
         * Returns the number of mappings. This method should be used
         * instead of {@link #size} because a ConcurrentHashMap may
         * contain more mappings than can be represented as an int. The
         * value returned is an estimate; the actual count may differ if
         * there are concurrent insertions or removals.
         *
         * @return the number of mappings
         * @since 1.8
         */
        public long mappingCount() {
            long n = sumCount();
            return (n < 0L) ? 0L : n; // ignore transient negative values
        }

         final long sumCount() {
            CounterCell[] as = counterCells; CounterCell a;
            long sum = baseCount;
            if (as != null) {
                for (int i = 0; i < as.length; ++i) {
                    if ((a = as[i]) != null)
                        sum += a.value;//所有counter的值求和
                }
            }
            return sum;
        }

### 2.8.3 addCount方法

在put方法结尾处调用了addCount方法，把当前ConcurrentHashMap的元素个数+1这个方法一共做了两件事,更新baseCount的值，检测是否进行扩容。

    private final void addCount(long x, int check) {
            CounterCell[] as; long b, s;
            //利用CAS方法更新baseCount的值 
            if ((as = counterCells) != null ||
                !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
                CounterCell a; long v; int m;
                boolean uncontended = true;
                if (as == null || (m = as.length - 1) < 0 ||
                    (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                    !(uncontended =
                      U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                    fullAddCount(x, uncontended);
                    return;
                }
                if (check <= 1)
                    return;
                s = sumCount();
            }
            //如果check值大于等于0 则需要检验是否需要进行扩容操作
            if (check >= 0) {
                Node<K,V>[] tab, nt; int n, sc;
                while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                       (n = tab.length) < MAXIMUM_CAPACITY) {
                    int rs = resizeStamp(n);
                    //
                    if (sc < 0) {
                        if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                            sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                            transferIndex <= 0)
                            break;
                         //如果已经有其他线程在执行扩容操作
                        if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                            transfer(tab, nt);
                    }
                    //当前线程是唯一的或是第一个发起扩容的线程  此时nextTable=null
                    else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                                 (rs << RESIZE_STAMP_SHIFT) + 2))
                        transfer(tab, null);
                    s = sumCount();
                }
            }
        }

## 总结

JDK6,7中的ConcurrentHashmap主要使用Segment来实现减小锁粒度，把HashMap分割成若干个Segment，在put的时候需要锁住Segment，get时候不加锁，使用volatile来保证可见性，当要统计全局时（比如size），首先会尝试多次计算modcount来确定，这几次尝试中，是否有其他线程进行了修改操作，如果没有，则直接返回size。如果有，则需要依次锁住所有的Segment来计算。

jdk7中ConcurrentHashmap中，当长度过长碰撞会很频繁，链表的增改删查操作都会消耗很长的时间，影响性能,所以jdk8 中完全重写了concurrentHashmap,代码量从原来的1000多行变成了 6000多 行，实现上也和原来的分段式存储有很大的区别。

主要设计上的变化有以下几点:

不采用segment而采用node，锁住node来实现减小锁粒度。
设计了MOVED状态 当resize的中过程中 线程2还在put数据，线程2会帮助resize。
使用3个CAS操作来确保node的一些操作的原子性，这种方式代替了锁。
sizeCtl的不同值来代表不同含义，起到了控制的作用。

至于为什么JDK8中使用synchronized而不是ReentrantLock，我猜是因为JDK8中对synchronized有了足够的优化吧。
