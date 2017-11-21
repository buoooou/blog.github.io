---
layout: post
published: false
title: java高并发下的HashMap
---
# 高并发下的HashMap

HashMap的容量是有限的。当经过多次元素插入，使得HashMap达到一定饱和度时，Key映射位置发生冲突的几率会逐渐提高。

这时候，HashMap需要扩展它的长度，也就是进行Resize。

影响发生Resize的因素有两个：

### 1.Capacity

HashMap的当前长度。上一期曾经说过，HashMap的长度是2的幂。

### 2.LoadFactor

HashMap负载因子，默认值为0.75f。

衡量HashMap是否进行Resize的条件如下：

HashMap.Size   >=  Capacity * LoadFactor


## 1.扩容

创建一个新的Entry空数组，长度是原数组的2倍。


## 2.ReHash

遍历原Entry数组，把所有的Entry重新Hash到新数组。为什么要重新Hash呢？因为长度扩大以后，Hash的规则也随之改变。

让我们回顾一下Hash公式：
index =  HashCode（Key） &  （Length - 1） 

当原数组长度为8时，Hash运算是和111B做与运算；新数组长度为16，Hash运算是和1111B做与运算。Hash结果显然不同。

Resize前的HashMap：


Resize后的HashMap：


ReHash的Java代码如下：

    /**
     * Transfers all entries from current table to newTable.
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }

注意：下面的内容十分烧脑，请小伙伴们坐稳扶好。


假设一个HashMap已经到了Resize的临界点。此时有两个线程A和B，在同一时刻对HashMap进行Put操作：

此时达到Resize条件，两个线程各自进行Rezie的第一步，也就是扩容：

这时候，两个线程都走到了ReHash的步骤。让我们回顾一下ReHash的代码：

假如此时线程B遍历到Entry3对象，刚执行完红框里的这行代码，线程就被挂起。对于线程B来说：

    e = Entry3
    next = Entry2

这时候线程A畅通无阻地进行着Rehash，当ReHash完成后，结果如下（图中的e和next，代表线程B的两个引用）：

直到这一步，看起来没什么毛病。接下来线程B恢复，继续执行属于它自己的ReHash。线程B刚才的状态是：

    e = Entry3
    next = Entry2

当执行到上面这一行时，显然 i = 3，因为刚才线程A对于Entry3的hash结果也是3。

我们继续执行到这两行，Entry3放入了线程B的数组下标为3的位置，并且e指向了Entry2。此时e和next的指向如下：

    e = Entry2
    next = Entry2

整体情况如图所示：

接着是新一轮循环，又执行到红框内的代码行：

    e = Entry2
    next = Entry3

接下来执行下面的三行，用头插法把Entry2插入到了线程B的数组的头结点：

第三次循环开始，又执行到红框的代码：

    e = Entry3
    next = Entry3.next = null

最后一步，当我们执行下面这一行的时候，见证奇迹的时刻来临了：

    newTable[i] = Entry2
    e = Entry3
    Entry2.next = Entry3
    Entry3.next = Entry2

链表出现了环形！


此时，问题还没有直接产生。当调用Get查找一个不存在的Key，而这个Key的Hash结果恰好等于3的时候，由于位置3带有环形链表，所以程序将会进入死循环！


1.Hashmap在插入元素过多的时候需要进行Resize，Resize的条件是
HashMap.Size   >=  Capacity * LoadFactor。

2.Hashmap的Resize包含扩容和ReHash两个步骤，ReHash在并发的情况下可能会形成链表环。
