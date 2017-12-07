---
layout: post
published: true
title: ArrayList 底层数组扩容原理
---
# ArrayList 底层数组扩容原理

System.arraycopy(）进行数组拷贝

我们知道当ArrayList如果不指定构造个数的话，第一次往里面添加元素时底层数组会初始化一个长度为10的数组，我们再回顾一下昨天的源码，再来看一下ArrayList里的源码，当添加第11个元素时

![a2663a19dcc7814bfcd45f6f894e3ec8.png]({{site.baseurl}}/img/a2663a19dcc7814bfcd45f6f894e3ec8.png)


再看grow()方法

![dfe9b7a69734c4f22ec66486c3170715.png]({{site.baseurl}}/img/dfe9b7a69734c4f22ec66486c3170715.png)


这儿有一段代码：int newCapacity = oldCapacity + (oldCapacity >> 1)，>>是移位运算符，相当于int newCapacity = oldCapacity + (oldCapacity/2)，但性能会好一些。

![e19ab9bccc0eb0327474e69c3212832e.png]({{site.baseurl}}/img/e19ab9bccc0eb0327474e69c3212832e.png)


本文开始那个问题，到这儿就解决了，这就是数组的扩容，一般是oldCapacity + (oldCapacity >> 1)，相当于扩容1.5倍。

看到这里，相信在以后的面试中，面试官再问数组和ArrayLIst的区别的时候，大家应该有了自己的理解，而不是去背面试题了。

ArrayList还提供了其它构造方法，我们顺便来看一下。

![e5f21a782bce9ba5eab702e4b15ab410.png]({{site.baseurl}}/img/e5f21a782bce9ba5eab702e4b15ab410.png)


我们再看一下源码，好简单：

![2c225955f5a838dc940a3d0d409ec9c7.png]({{site.baseurl}}/img/2c225955f5a838dc940a3d0d409ec9c7.png)


当我们在写代码过程中，如果我们大概知道元素的个数，比如一个班级大概有40-50人，我们优先考虑List<Person> list2 = new ArrayList<>(50)以指定个数的方式去构造，这样可以避免底层数组的多次拷贝，进而提高程序性能。
  