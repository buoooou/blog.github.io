---
layout: post
published: true
title: ConcurrentHashMap 总结（ 中 ）
---
# ConcurrentHashMap 总结（ 中 ）

    /**
        * 一个过渡的table表  只有在扩容的时候才会使用
        */
       private transient volatile Node<K,V>[] nextTable;

    /**
        * Moves and/or copies the nodes in each bin to new table. See
        * above for explanation.
        */
       private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
           int n = tab.length, stride;
           if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
               stride = MIN_TRANSFER_STRIDE; // subdivide range
           if (nextTab == null) {            // initiating
               try {
                   @SuppressWarnings("unchecked")
                   Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];//构造一个nextTable对象 它的容量是原来的两倍
                   nextTab = nt;
               } catch (Throwable ex) {      // try to cope with OOME
                   sizeCtl = Integer.MAX_VALUE;
                   return;
               }
               nextTable = nextTab;
               transferIndex = n;
           }
           int nextn = nextTab.length;
           ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);//构造一个连节点指针 用于标志位
           boolean advance = true;//并发扩容的关键属性 如果等于true 说明这个节点已经处理过
           boolean finishing = false; // to ensure sweep before committing nextTab
           for (int i = 0, bound = 0;;) {
               Node<K,V> f; int fh;
               //这个while循环体的作用就是在控制i--  通过i--可以依次遍历原hash表中的节点
               while (advance) {
                   int nextIndex, nextBound;
                   if (--i >= bound || finishing)
                       advance = false;
                   else if ((nextIndex = transferIndex) <= 0) {
                       i = -1;
                       advance = false;
                   }
                   else if (U.compareAndSwapInt
                            (this, TRANSFERINDEX, nextIndex,
                             nextBound = (nextIndex > stride ?
                                          nextIndex - stride : 0))) {
                       bound = nextBound;
                       i = nextIndex - 1;
                       advance = false;
                   }
               }
               if (i < 0 || i >= n || i + n >= nextn) {
                   int sc;
                   if (finishing) {
                       //如果所有的节点都已经完成复制工作  就把nextTable赋值给table 清空临时对象nextTable
                       nextTable = null;
                       table = nextTab;
                       sizeCtl = (n << 1) - (n >>> 1);//扩容阈值设置为原来容量的1.5倍  依然相当于现在容量的0.75倍
                       return;
                   }
                   //利用CAS方法更新这个扩容阈值，在这里面sizectl值减一，说明新加入一个线程参与到扩容操作
                   if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                       if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                           return;
                       finishing = advance = true;
                       i = n; // recheck before commit
                   }
               }
               //如果遍历到的节点为空 则放入ForwardingNode指针
               else if ((f = tabAt(tab, i)) == null)
                   advance = casTabAt(tab, i, null, fwd);
               //如果遍历到ForwardingNode节点  说明这个点已经被处理过了 直接跳过  这里是控制并发扩容的核心
               else if ((fh = f.hash) == MOVED)
                   advance = true; // already processed
               else {
                       //节点上锁
                   synchronized (f) {
                       if (tabAt(tab, i) == f) {
                           Node<K,V> ln, hn;
                           //如果fh>=0 证明这是一个Node节点
                           if (fh >= 0) {
                               int runBit = fh & n;
                               //以下的部分在完成的工作是构造两个链表  一个是原链表  另一个是原链表的反序排列
                               Node<K,V> lastRun = f;
                               for (Node<K,V> p = f.next; p != null; p = p.next) {
                                   int b = p.hash & n;
                                   if (b != runBit) {
                                       runBit = b;
                                       lastRun = p;
                                   }
                               }
                               if (runBit == 0) {
                                   ln = lastRun;
                                   hn = null;
                               }
                               else {
                                   hn = lastRun;
                                   ln = null;
                               }
                               for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                   int ph = p.hash; K pk = p.key; V pv = p.val;
                                   if ((ph & n) == 0)
                                       ln = new Node<K,V>(ph, pk, pv, ln);
                                   else
                                       hn = new Node<K,V>(ph, pk, pv, hn);
                               }
                               //在nextTable的i位置上插入一个链表
                               setTabAt(nextTab, i, ln);
                               //在nextTable的i+n的位置上插入另一个链表
                               setTabAt(nextTab, i + n, hn);
                               //在table的i位置上插入forwardNode节点  表示已经处理过该节点
                               setTabAt(tab, i, fwd);
                               //设置advance为true 返回到上面的while循环中 就可以执行i--操作
                               advance = true;
                           }
                           //对TreeBin对象进行处理  与上面的过程类似
                           else if (f instanceof TreeBin) {
                               TreeBin<K,V> t = (TreeBin<K,V>)f;
                               TreeNode<K,V> lo = null, loTail = null;
                               TreeNode<K,V> hi = null, hiTail = null;
                               int lc = 0, hc = 0;
                               //构造正序和反序两个链表
                               for (Node<K,V> e = t.first; e != null; e = e.next) {
                                   int h = e.hash;
                                   TreeNode<K,V> p = new TreeNode<K,V>
                                       (h, e.key, e.val, null, null);
                                   if ((h & n) == 0) {
                                       if ((p.prev = loTail) == null)
                                           lo = p;
                                       else
                                           loTail.next = p;
                                       loTail = p;
                                       ++lc;
                                   }
                                   else {
                                       if ((p.prev = hiTail) == null)
                                           hi = p;
                                       else
                                           hiTail.next = p;
                                       hiTail = p;
                                       ++hc;
                                   }
                               }
                               //如果扩容后已经不再需要tree的结构 反向转换为链表结构
                               ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                   (hc != 0) ? new TreeBin<K,V>(lo) : t;
                               hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                   (lc != 0) ? new TreeBin<K,V>(hi) : t;
                                //在nextTable的i位置上插入一个链表    
                               setTabAt(nextTab, i, ln);
                               //在nextTable的i+n的位置上插入另一个链表
                               setTabAt(nextTab, i + n, hn);
                                //在table的i位置上插入forwardNode节点  表示已经处理过该节点
                               setTabAt(tab, i, fwd);
                               //设置advance为true 返回到上面的while循环中 就可以执行i--操作
                               advance = true;
                           }
                       }
                   }
               }
           }
       }

## 2.6 Put方法

前面的所有的介绍其实都为这个方法做铺垫。ConcurrentHashMap最常用的就是put和get两个方法。现在来介绍put方法，这个put方法依然沿用HashMap的put方法的思想，根据hash值计算这个新插入的点在table中的位置i，如果i位置是空的，直接放进去，否则进行判断，如果i位置是树节点，按照树的方式插入新的节点，否则把i插入到链表的末尾。ConcurrentHashMap中依然沿用这个思想，有一个最重要的不同点就是ConcurrentHashMap不允许key或value为null值。另外由于涉及到多线程，put方法就要复杂一点。在多线程中可能有以下两个情况

如果一个或多个线程正在对ConcurrentHashMap进行扩容操作，当前线程也要进入扩容的操作中。这个扩容的操作之所以能被检测到，是因为transfer方法中在空结点上插入forward节点，如果检测到需要插入的位置被forward节点占有，就帮助进行扩容；
如果检测到要插入的节点是非空且不是forward节点，就对这个节点加锁，这样就保证了线程安全。尽管这个有一些影响效率，但是还是会比hashTable的synchronized要好得多。

整体流程就是首先定义不允许key或value为null的情况放入  对于每一个放入的值，首先利用spread方法对key的hashcode进行一次hash计算，由此来确定这个值在table中的位置。

如果这个位置是空的，那么直接放入，而且不需要加锁操作。

如果这个位置存在结点，说明发生了hash碰撞，首先判断这个节点的类型。如果是链表节点（fh>0）,则得到的结点就是hash值相同的节点组成的链表的头节点。需要依次向后遍历确定这个新加入的值所在位置。如果遇到hash值与key值都与新加入节点是一致的情况，则只需要更新value值即可。否则依次向后遍历，直到链表尾插入这个结点。如果加入这个节点以后链表长度大于8，就把这个链表转换成红黑树。如果这个节点的类型已经是树节点的话，直接调用树节点的插入方法进行插入新的值。

    public V put(K key, V value) {
            return putVal(key, value, false);
        }

        /** Implementation for put and putIfAbsent */
        final V putVal(K key, V value, boolean onlyIfAbsent) {
                //不允许 key或value为null
            if (key == null || value == null) throw new NullPointerException();
            //计算hash值
            int hash = spread(key.hashCode());
            int binCount = 0;
            //死循环 何时插入成功 何时跳出
            for (Node<K,V>[] tab = table;;) {
                Node<K,V> f; int n, i, fh;
                //如果table为空的话，初始化table
                if (tab == null || (n = tab.length) == 0)
                    tab = initTable();
                //根据hash值计算出在table里面的位置 
                else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                    //如果这个位置没有值 ，直接放进去，不需要加锁
                    if (casTabAt(tab, i, null,
                                 new Node<K,V>(hash, key, value, null)))
                        break;                   // no lock when adding to empty bin
                }
                //当遇到表连接点时，需要进行整合表的操作
                else if ((fh = f.hash) == MOVED)
                    tab = helpTransfer(tab, f);
                else {
                    V oldVal = null;
                    //结点上锁  这里的结点可以理解为hash值相同组成的链表的头结点
                    synchronized (f) {
                        if (tabAt(tab, i) == f) {
                            //fh〉0 说明这个节点是一个链表的节点 不是树的节点
                            if (fh >= 0) {
                                binCount = 1;
                                //在这里遍历链表所有的结点
                                for (Node<K,V> e = f;; ++binCount) {
                                    K ek;
                                    //如果hash值和key值相同  则修改对应结点的value值
                                    if (e.hash == hash &&
                                        ((ek = e.key) == key ||
                                         (ek != null && key.equals(ek)))) {
                                        oldVal = e.val;
                                        if (!onlyIfAbsent)
                                            e.val = value;
                                        break;
                                    }
                                    Node<K,V> pred = e;
                                    //如果遍历到了最后一个结点，那么就证明新的节点需要插入 就把它插入在链表尾部
                                    if ((e = e.next) == null) {
                                        pred.next = new Node<K,V>(hash, key,
                                                                  value, null);
                                        break;
                                    }
                                }
                            }
                            //如果这个节点是树节点，就按照树的方式插入值
                            else if (f instanceof TreeBin) {
                                Node<K,V> p;
                                binCount = 2;
                                if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                               value)) != null) {
                                    oldVal = p.val;
                                    if (!onlyIfAbsent)
                                        p.val = value;
                                }
                            }
                        }
                    }
                    if (binCount != 0) {
                        //如果链表长度已经达到临界值8 就需要把链表转换为树结构
                        if (binCount >= TREEIFY_THRESHOLD)
                            treeifyBin(tab, i);
                        if (oldVal != null)
                            return oldVal;
                        break;
                    }
                }
            }
            //将当前ConcurrentHashMap的元素数量+1
            addCount(1L, binCount);
            return null;
        }

我们可以发现JDK8中的实现也是锁分离的思想，只是锁住的是一个Node，而不是JDK7中的Segment，而锁住Node之前的操作是无锁的并且也是线程安全的，建立在之前提到的3个原子操作上。

### 2.6.1 helpTransfer方法

这是一个协助扩容的方法。这个方法被调用的时候，当前ConcurrentHashMap一定已经有了nextTable对象，首先拿到这个nextTable对象，调用transfer方法。回看上面的transfer方法可以看到，当本线程进入扩容方法的时候会直接进入复制阶段。

### 2.6.2 treeifyBin方法

这个方法用于将过长的链表转换为TreeBin对象。但是他并不是直接转换，而是进行一次容量判断，如果容量没有达到转换的要求，直接进行扩容操作并返回；如果满足条件才链表的结构抓换为TreeBin ，这与HashMap不同的是，它并没有把TreeNode直接放入红黑树，而是利用了TreeBin这个小容器来封装所有的TreeNode.

## 2.7 get方法

get方法比较简单，给定一个key来确定value的时候，必须满足两个条件  key相同  hash值相同，对于节点可能在链表或树上的情况，需要分别去查找。

    public V get(Object key) {
            Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
            //计算hash值
            int h = spread(key.hashCode());
            //根据hash值确定节点位置
            if ((tab = table) != null && (n = tab.length) > 0 &&
                (e = tabAt(tab, (n - 1) & h)) != null) {
                //如果搜索到的节点key与传入的key相同且不为null,直接返回这个节点  
                if ((eh = e.hash) == h) {
                    if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                        return e.val;
                }
                //如果eh<0 说明这个节点在树上 直接寻找
                else if (eh < 0)
                    return (p = e.find(h, key)) != null ? p.val : null;
                 //否则遍历链表 找到对应的值并返回
                while ((e = e.next) != null) {
                    if (e.hash == h &&
                        ((ek = e.key) == key || (ek != null && key.equals(ek))))
                        return e.val;
                }
            }
            return null;
        }
