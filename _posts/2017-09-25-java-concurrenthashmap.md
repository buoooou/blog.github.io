---
layout: post
published: true
title: Java ConcurrentHashMap 总结
---
# Java ConcurrentHashMap 总结

并发编程实践中，ConcurrentHashMap是一个经常被使用的数据结构，相比于Hashtable以及Collections.synchronizedMap()，ConcurrentHashMap在线程安全的基础上提供了更好的写并发能力，但同时降低了对读一致性的要求（这点好像CAP理论啊 O(∩_∩)O）。ConcurrentHashMap的设计与实现非常精巧，大量的利用了volatile，final，CAS等lock-free技术来减少锁竞争对于性能的影响，无论对于Java并发编程的学习还是Java内存模型的理解，ConcurrentHashMap的设计以及源码都值得非常仔细的阅读与揣摩。

这篇日志记录了自己对ConcurrentHashMap的一些总结，由于JDK6，7，8中实现都不同，需要分开阐述在不同版本中的ConcurrentHashMap。


## 1. JDK6与JDK7中的实现

### 1.1 设计思路

ConcurrentHashMap采用了分段锁的设计，只有在同一个分段内才存在竞态关系，不同的分段锁之间没有锁竞争。相比于对整个Map加锁的设计，分段锁大大的提高了高并发环境下的处理能力。但同时，由于不是对整个Map加锁，导致一些需要扫描整个Map的方法（如size(), containsValue()）需要使用特殊的实现，另外一些方法（如clear()）甚至放弃了对一致性的要求（ConcurrentHashMap是弱一致性的，[具体请查看ConcurrentHashMap能完全替代HashTable吗？](http://my.oschina.net/hosee/blog/675423)）。


ConcurrentHashMap中的分段锁称为Segment，它即类似于HashMap（[JDK7与JDK8中HashMap的实现](http://my.oschina.net/hosee/blog/618953)）的结构，即内部拥有一个Entry数组，数组中的每个元素又是一个链表；同时又是一个ReentrantLock（Segment继承了ReentrantLock）。ConcurrentHashMap中的HashEntry相对于HashMap中的Entry有一定的差异性：HashEntry中的value以及next都被volatile修饰，这样在多线程读写过程中能够保持它们的可见性，代码如下：

    static final class HashEntry<K,V> {
            final int hash;
            final K key;
            volatile V value;
            volatile HashEntry<K,V> next;

### 1.2 并发度（Concurrency Level）

并发度可以理解为程序运行时能够同时更新ConccurentHashMap且不产生锁竞争的最大线程数，实际上就是ConcurrentHashMap中的分段锁个数，即Segment[]的数组长度。ConcurrentHashMap默认的并发度为16，但用户也可以在构造函数中设置并发度。当用户设置并发度时，ConcurrentHashMap会使用大于等于该值的最小2幂指数作为实际并发度（假如用户设置并发度为17，实际并发度则为32）。运行时通过将key的高n位（n = 32 – segmentShift）和并发度减1（segmentMask）做位与运算定位到所在的Segment。segmentShift与segmentMask都是在构造过程中根据concurrency level被相应的计算出来。

如果并发度设置的过小，会带来严重的锁竞争问题；如果并发度设置的过大，原本位于同一个Segment内的访问会扩散到不同的Segment中，CPU cache命中率会下降，从而引起程序性能下降。（文档的说法是根据你并发的线程数量决定，太多会导性能降低）

### 1.3 创建分段锁

和JDK6不同，JDK7中除了第一个Segment之外，剩余的Segments采用的是延迟初始化的机制：每次put之前都需要检查key对应的Segment是否为null，如果是则调用ensureSegment()以确保对应的Segment被创建。

ensureSegment可能在并发环境下被调用，但与想象中不同，ensureSegment并未使用锁来控制竞争，而是使用了Unsafe对象的getObjectVolatile()提供的原子读语义结合CAS来确保Segment创建的原子性。代码段如下：

    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                    == null) { // recheck
                    Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
                    while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                           == null) {
                        if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                            break;
                    }
    }

### 1.4 put/putIfAbsent/putAll

和JDK6一样，ConcurrentHashMap的put方法被代理到了对应的Segment（定位Segment的原理之前已经描述过）中。与JDK6不同的是，JDK7版本的ConcurrentHashMap在获得Segment锁的过程中，做了一定的优化 - 在真正申请锁之前，put方法会通过tryLock()方法尝试获得锁，在尝试获得锁的过程中会对对应hashcode的链表进行遍历，如果遍历完毕仍然找不到与key相同的HashEntry节点，则为后续的put操作提前创建一个HashEntry。当tryLock一定次数后仍无法获得锁，则通过lock申请锁。

需要注意的是，由于在并发环境下，其他线程的put，rehash或者remove操作可能会导致链表头结点的变化，因此在过程中需要进行检查，如果头结点发生变化则重新对表进行遍历。而如果其他线程引起了链表中的某个节点被删除，即使该变化因为是非原子写操作（删除节点后链接后续节点调用的是Unsafe.putOrderedObject()，该方法不提供原子写语义）可能导致当前线程无法观察到，但因为不影响遍历的正确性所以忽略不计。

之所以在获取锁的过程中对整个链表进行遍历，主要目的是希望遍历的链表被CPU cache所缓存，为后续实际put过程中的链表遍历操作提升性能。

在获得锁之后，Segment对链表进行遍历，如果某个HashEntry节点具有相同的key，则更新该HashEntry的value值，否则新建一个HashEntry节点，将它设置为链表的新head节点并将原头节点设为新head的下一个节点。新建过程中如果节点总数（含新建的HashEntry）超过threshold，则调用rehash()方法对Segment进行扩容，最后将新建HashEntry写入到数组中。

put方法中，链接新节点的下一个节点（HashEntry.setNext()）以及将链表写入到数组中（setEntryAt()）都是通过Unsafe的putOrderedObject()方法来实现，这里并未使用具有原子写语义的putObjectVolatile()的原因是：JMM会保证获得锁到释放锁之间所有对象的状态更新都会在锁被释放之后更新到主存，从而保证这些变更对其他线程是可见的。

### 1.5 rehash

相对于HashMap的resize，ConcurrentHashMap的rehash原理类似，但是Doug Lea为rehash做了一定的优化，避免让所有的节点都进行复制操作：由于扩容是基于2的幂指来操作，假设扩容前某HashEntry对应到Segment中数组的index为i，数组的容量为capacity，那么扩容后该HashEntry对应到新数组中的index只可能为i或者i+capacity，因此大多数HashEntry节点在扩容前后index可以保持不变。基于此，rehash方法中会定位第一个后续所有节点在扩容后index都保持不变的节点，然后将这个节点之前的所有节点重排即可。这部分代码如下：

    private void rehash(HashEntry<K,V> node) {
               HashEntry<K,V>[] oldTable = table;
               int oldCapacity = oldTable.length;
               int newCapacity = oldCapacity << 1;
               threshold = (int)(newCapacity * loadFactor);
               HashEntry<K,V>[] newTable =
                   (HashEntry<K,V>[]) new HashEntry[newCapacity];
               int sizeMask = newCapacity - 1;
               for (int i = 0; i < oldCapacity ; i++) {
                   HashEntry<K,V> e = oldTable[i];
                   if (e != null) {
                       HashEntry<K,V> next = e.next;
                       int idx = e.hash & sizeMask;
                       if (next == null)   //  Single node on list
                           newTable[idx] = e;
                       else { // Reuse consecutive sequence at same slot
                           HashEntry<K,V> lastRun = e;
                           int lastIdx = idx;
                           for (HashEntry<K,V> last = next;
                                last != null;
                                last = last.next) {
                               int k = last.hash & sizeMask;
                               if (k != lastIdx) {
                                   lastIdx = k;
                                   lastRun = last;
                               }
                           }
                           newTable[lastIdx] = lastRun;
                           // Clone remaining nodes
                           for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                               V v = p.value;
                               int h = p.hash;
                               int k = h & sizeMask;
                               HashEntry<K,V> n = newTable[k];
                               newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                           }
                       }
                   }
               }
               int nodeIndex = node.hash & sizeMask; // add the new node
               node.setNext(newTable[nodeIndex]);
               newTable[nodeIndex] = node;
               table = newTable;
           }

### 1.6 remove

和put类似，remove在真正获得锁之前，也会对链表进行遍历以提高缓存命中率。

### 1.7 get与containsKey

get与containsKey两个方法几乎完全一致：他们都没有使用锁，而是通过Unsafe对象的getObjectVolatile()方法提供的原子读语义，来获得Segment以及对应的链表，然后对链表遍历判断是否存在key相同的节点以及获得该节点的value。但由于遍历过程中其他线程可能对链表结构做了调整，因此get和containsKey返回的可能是过时的数据，这一点是ConcurrentHashMap在弱一致性上的体现。如果要求强一致性，那么必须使用Collections.synchronizedMap()方法。

### 1.8 size、containsValue

这些方法都是基于整个ConcurrentHashMap来进行操作的，他们的原理也基本类似：首先不加锁循环执行以下操作：循环所有的Segment（通过Unsafe的getObjectVolatile()以保证原子读语义），获得对应的值以及所有Segment的modcount之和。如果连续两次所有Segment的modcount和相等，则过程中没有发生其他线程修改ConcurrentHashMap的情况，返回获得的值。

当循环次数超过预定义的值时，这时需要对所有的Segment依次进行加锁，获取返回值后再依次解锁。值得注意的是，加锁过程中要强制创建所有的Segment，否则容易出现其他线程创建Segment并进行put，remove等操作。代码如下：

    for(int j =0; j < segments.length; ++j)

    ensureSegment(j).lock();// force creation

一般来说，应该避免在多线程环境下使用size和containsValue方法。

注1：modcount在put, replace, remove以及clear等方法中都会被修改。

注2：对于containsValue方法来说，如果在循环过程中发现匹配value的HashEntry，则直接返回true。

最后，与HashMap不同的是，ConcurrentHashMap并不允许key或者value为null，按照Doug Lea的说法，这么设计的原因是在ConcurrentHashMap中，一旦value出现null，则代表HashEntry的key/value没有映射完成就被其他线程所见，需要特殊处理。在JDK6中，get方法的实现中就有一段对HashEntry.value == null的防御性判断。但Doug Lea也承认实际运行过程中，这种情况似乎不可能发生（参考：http://cs.oswego.edu/pipermail/concurrency-interest/2011-March/007799.html）。

## 2. JDK8中的实现

ConcurrentHashMap在JDK8中进行了巨大改动，很需要通过源码来再次学习下Doug Lea的实现方法。

它摒弃了Segment（锁段）的概念，而是启用了一种全新的方式实现,利用CAS算法。它沿用了与它同时期的HashMap版本的思想，底层依然由“数组”+链表+红黑树的方式思想(JDK7与JDK8中HashMap的实现)，但是为了做到并发，又增加了很多辅助的类，例如TreeBin，Traverser等对象内部类。

### 2.1 重要的属性

首先来看几个重要的属性，与HashMap相同的就不再介绍了，这里重点解释一下sizeCtl这个属性。可以说它是ConcurrentHashMap中出镜率很高的一个属性，因为它是一个控制标识符，在不同的地方有不同用途，而且它的取值不同，也代表不同的含义。

负数代表正在进行初始化或扩容操作
-1代表正在初始化
-N 表示有N-1个线程正在进行扩容操作
正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，这一点类似于扩容阈值的概念。还后面可以看到，它的值始终是当前ConcurrentHashMap容量的0.75倍，这与loadfactor是对应的。

    	/**
         * 盛装Node元素的数组 它的大小是2的整数次幂
         * Size is always a power of two. Accessed directly by iterators.
         */
        transient volatile Node<K,V>[] table;

            /**
         * Table initialization and resizing control.  When negative, the
         * table is being initialized or resized: -1 for initialization,
         * else -(1 + the number of active resizing threads).  Otherwise,
         * when table is null, holds the initial table size to use upon
         * creation, or 0 for default. After initialization, holds the
         * next element count value upon which to resize the table.
         hash表初始化或扩容时的一个控制位标识量。
         负数代表正在进行初始化或扩容操作
         -1代表正在初始化
         -N 表示有N-1个线程正在进行扩容操作
         正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小

         */
        private transient volatile int sizeCtl; 
        // 以下两个是用来控制扩容的时候 单线程进入的变量
         /**
         * The number of bits used for generation stamp in sizeCtl.
         * Must be at least 6 for 32bit arrays.
         */
        private static int RESIZE_STAMP_BITS = 16;
            /**
         * The bit shift for recording size stamp in sizeCtl.
         */
        private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

        /*
         * Encodings for Node hash fields. See above for explanation.
         */
        static final int MOVED     = -1; // hash值是-1，表示这是一个forwardNode节点
        static final int TREEBIN   = -2; // hash值是-2  表示这时一个TreeBin节点

### 2.2 重要的类

**2.2.1 Node**

Node是最核心的内部类，它包装了key-value键值对，所有插入ConcurrentHashMap的数据都包装在这里面。它与HashMap中的定义很相似，但是但是有一些差别它对value和next属性设置了volatile同步锁(与JDK7的Segment相同)，它不允许调用setValue方法直接改变Node的value域，它增加了find方法辅助map.get()方法。

**2.2.2 TreeNode**

树节点类，另外一个核心的数据结构。当链表长度过长的时候，会转换为TreeNode。但是与HashMap不相同的是，它并不是直接转换为红黑树，而是把这些结点包装成TreeNode放在TreeBin对象中，由TreeBin完成对红黑树的包装。而且TreeNode在ConcurrentHashMap集成自Node类，而并非HashMap中的集成自LinkedHashMap.Entry<K,V>类，也就是说TreeNode带有next指针，这样做的目的是方便基于TreeBin的访问。

**2.2.3 TreeBin**

这个类并不负责包装用户的key、value信息，而是包装的很多TreeNode节点。它代替了TreeNode的根节点，也就是说在实际的ConcurrentHashMap“数组”中，存放的是TreeBin对象，而不是TreeNode对象，这是与HashMap的区别。另外这个类还带有了读写锁。

这里仅贴出它的构造方法。可以看到在构造TreeBin节点时，仅仅指定了它的hash值为TREEBIN常量，这也就是个标识为。同时也看到我们熟悉的红黑树构造方法

**2.2.4 ForwardingNode**

一个用于连接两个table的节点类。它包含一个nextTable指针，用于指向下一张表。而且这个节点的key value next指针全部为null，它的hash值为-1. 这里面定义的find的方法是从nextTable里进行查询节点，而不是以自身为头节点进行查找。

	/**
     * A node inserted at head of bins during transfer operations.
     */
    static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }
 
        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            outer: for (Node<K,V>[] tab = nextTable;;) {
                Node<K,V> e; int n;
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;
                for (;;) {
                    int eh; K ek;
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    if (eh < 0) {
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            return e.find(h, k);
                    }
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
    }

### 2.3 Unsafe与CAS

在ConcurrentHashMap中，随处可以看到U, 大量使用了U.compareAndSwapXXX的方法，这个方法是利用一个CAS算法实现无锁化的修改值的操作，他可以大大降低锁代理的性能消耗。这个算法的基本思想就是不断地去比较当前内存中的变量值与你指定的一个变量值是否相等，如果相等，则接受你指定的修改的值，否则拒绝你的操作。因为当前线程中的值已经不是最新的值，你的修改很可能会覆盖掉其他线程修改的结果。这一点与乐观锁，SVN的思想是比较类似的。

**2.3.1 unsafe静态块**

unsafe代码块控制了一些属性的修改工作，比如最常用的SIZECTL 。在这一版本的concurrentHashMap中，大量应用来的CAS方法进行变量、属性的修改工作。利用CAS进行无锁操作，可以大大提高性能。

    private static final sun.misc.Unsafe U;
       private static final long SIZECTL;
       private static final long TRANSFERINDEX;
       private static final long BASECOUNT;
       private static final long CELLSBUSY;
       private static final long CELLVALUE;
       private static final long ABASE;
       private static final int ASHIFT;

       static {
           try {
               U = sun.misc.Unsafe.getUnsafe();
               Class<?> k = ConcurrentHashMap.class;
               SIZECTL = U.objectFieldOffset
                   (k.getDeclaredField("sizeCtl"));
               TRANSFERINDEX = U.objectFieldOffset
                   (k.getDeclaredField("transferIndex"));
               BASECOUNT = U.objectFieldOffset
                   (k.getDeclaredField("baseCount"));
               CELLSBUSY = U.objectFieldOffset
                   (k.getDeclaredField("cellsBusy"));
               Class<?> ck = CounterCell.class;
               CELLVALUE = U.objectFieldOffset
                   (ck.getDeclaredField("value"));
               Class<?> ak = Node[].class;
               ABASE = U.arrayBaseOffset(ak);
               int scale = U.arrayIndexScale(ak);
               if ((scale & (scale - 1)) != 0)
                   throw new Error("data type scale not a power of two");
               ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
           } catch (Exception e) {
               throw new Error(e);
           }
       }

**2.3.2 三个核心方法**

ConcurrentHashMap定义了三个原子操作，用于对指定位置的节点进行操作。正是这些原子操作保证了ConcurrentHashMap的线程安全。

//获得在i位置上的Node节点
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
        //利用CAS算法设置i位置上的Node节点。之所以能实现并发是因为他指定了原来这个节点的值是多少
        //在CAS算法中，会比较内存中的值与你指定的这个值是否相等，如果相等才接受你的修改，否则拒绝你的修改
        //因此当前线程中的值并不是最新的值，这种修改可能会覆盖掉其他线程的修改结果  有点类似于SVN
        
        static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                            Node<K,V> c, Node<K,V> v) {
            return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
        }
            //利用volatile方法设置节点位置的值
        static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
            U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
        }

### 2.4 初始化方法initTable

对于ConcurrentHashMap来说，调用它的构造方法仅仅是设置了一些参数而已。而整个table的初始化是在向ConcurrentHashMap中插入元素的时候发生的。如调用put、computeIfAbsent、compute、merge等方法的时候，调用时机是检查table==null。

初始化方法主要应用了关键属性sizeCtl 如果这个值〈0，表示其他线程正在进行初始化，就放弃这个操作。在这也可以看出ConcurrentHashMap的初始化只能由一个线程完成。如果获得了初始化权限，就用CAS方法将sizeCtl置为-1，防止其他线程进入。初始化数组后，将sizeCtl的值改为0.75*n。

	/**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
                //sizeCtl表示有其他线程正在进行初始化操作，把线程挂起。对于table的初始化工作，只能有一个线程在进行。
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {//利用CAS方法把sizectl的值置为-1 表示本线程正在进行初始化
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);//相当于0.75*n 设置一个扩容的阈值
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }

### 2.5 扩容方法 transfer

当ConcurrentHashMap容量不足的时候，需要对table进行扩容。这个方法的基本思想跟HashMap是很像的，但是由于它是支持并发扩容的，所以要复杂的多。原因是它支持多线程进行扩容操作，而并没有加锁。我想这样做的目的不仅仅是为了满足concurrent的要求，而是希望利用并发处理去减少扩容带来的时间影响。因为在扩容的时候，总是会涉及到从一个“数组”到另一个“数组”拷贝的操作，如果这个操作能够并发进行，那真真是极好的了。

整个扩容操作分为两个部分

 第一部分是构建一个nextTable,它的容量是原来的两倍，这个操作是单线程完成的。这个单线程的保证是通过RESIZE_STAMP_SHIFT这个常量经过一次运算来保证的，这个地方在后面会有提到；
第二个部分就是将原来table中的元素复制到nextTable中，这里允许多线程进行操作。

先来看一下单线程是如何完成的：

它的大体思想就是遍历、复制的过程。首先根据运算得到需要遍历的次数i，然后利用tabAt方法获得i位置的元素：

如果这个位置为空，就在原table中的i位置放入forwardNode节点，这个也是触发并发扩容的关键点；
如果这个位置是Node节点（fh>=0），如果它是一个链表的头节点，就构造一个反序链表，把他们分别放在nextTable的i和i+n的位置上
如果这个位置是TreeBin节点（fh<0），也做一个反序处理，并且判断是否需要untreefi，把处理的结果分别放在nextTable的i和i+n的位置上
遍历过所有的节点以后就完成了复制工作，这时让nextTable作为新的table，并且更新sizeCtl为新容量的0.75倍 ，完成扩容。

再看一下多线程是如何完成的：

在代码的69行有一个判断，如果遍历到的节点是forward节点，就向后继续遍历，再加上给节点上锁的机制，就完成了多线程的控制。多线程遍历节点，处理了一个节点，就把对应点的值set为forward，另一个线程看到forward，就向后遍历。这样交叉就完成了复制工作。而且还很好的解决了线程安全的问题。 这个方法的设计实在是让我膜拜。
