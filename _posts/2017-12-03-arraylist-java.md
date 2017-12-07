---
layout: post
published: true
title: ArrayList初始化 - Java
---
# ArrayList初始化 - Java

首先ArrayList是一个普通的类，我们来看一段代码：

![90f7c9c4fefdc495841f6fba6b7532a2.png]({{site.baseurl}}/img/90f7c9c4fefdc495841f6fba6b7532a2.png)

首先：执行List<Person> list1 = new ArrayList<>();当看到new这个关键字的时候，我们脑袋里应该第一印象就是这货在堆内存开辟了一块空间，好我们再来画一画。

![c342b7bc7162de6ee6215b68586afa74.png]({{site.baseurl}}/img/c342b7bc7162de6ee6215b68586afa74.png)


注：常量池位于方法区，方法区位于堆内存，前面没涉及到，所以没画方法区，现在补上
好，既然是new出来的，那我们直接从构造函数入手，看一下构造函数做了什么。

![9911948739df321c2ab818857542bbca.png]({{site.baseurl}}/img/9911948739df321c2ab818857542bbca.png)


很简单，就一行代码，继续看一下，this.elementData和DEFAULTCAPACITY_EMPTY_ELEMENTDATA分别是什么

![82d373873afaae6eeb90e03ae99a4856.png]({{site.baseurl}}/img/82d373873afaae6eeb90e03ae99a4856.png)


红框里的内容是不是似曾相识？是的，和String一样，底层是数组，唯一的区别是String底层是char[]数组（忘了的可以复习一下，传送门：String是一个很普通的类 - Java那些事儿），而这儿是Object[]数组，也就是说该数组可以放任何对象（所有对象都继承自父类Object）,执行完构造函数后，如下图。

![af500c344720f39202d2236a5cbb940b.png]({{site.baseurl}}/img/af500c344720f39202d2236a5cbb940b.png)

注：static修饰的变量，常驻于方法区，我们不需要new，JVM会提前给我们初始化好，这个特性在实际开发过程中，经常拿来做缓存。在让人疑惑的Java代码 - Java那些事儿 一文中，我们文中Integer的缓存就是最好的例子。static变量又叫类变量，不管该类有多少个对象，static的变量只有一份，独一无二。
fianl修饰的变量，JVM也会提前给我们初始化好。
transient这个关键字告诉我们该对象在序列化的时候请忽略这个元素，后续我们会讲序列化，这儿先跳过。
继续执行：List<Person> list2 = new ArrayList<>();

![f81252075c4a58aa10d53664c1411702.png]({{site.baseurl}}/img/f81252075c4a58aa10d53664c1411702.png)


ArrayList这个类的作者真是好贴心，new的时候连缓存都考虑到了，为了避免我们反复的创建无用数组，所有新new出来的ArrayList底层数组都指向缓存在方法区里的Object[]数组。

继续执行Person person1 = new Person("张三")

![8780dcd8d1789233d538d03e841189d8.png]({{site.baseurl}}/img/8780dcd8d1789233d538d03e841189d8.png)


继续，执行list1.add(person1)，不多说，看源码ArrayList是怎么处理add的。

![2ccd4d3fd04624c65af4d3700335d582.png]({{site.baseurl}}/img/2ccd4d3fd04624c65af4d3700335d582.png)


我们先看ensureCapacityInternal方法，方法里有个参数是size，看们先看一下这个size从哪来的。

![ca9151ecf3667272a95c0820997ffd84.png]({{site.baseurl}}/img/ca9151ecf3667272a95c0820997ffd84.png)


原来是一个成员变量，相信大家看到size一猜就知道大概是干嘛的了吧。好，我们在图里的ArrayList对象里补上它，size是int基本数据类型，成员变量初始化的为0。

![898c4db8f00b230c803603a72c0e73d9.png]({{site.baseurl}}/img/898c4db8f00b230c803603a72c0e73d9.png)

继续往下看

![29dad3aafc9ccc857afcac5d27764cfe.png]({{site.baseurl}}/img/29dad3aafc9ccc857afcac5d27764cfe.png)


ensureCapacityInternal方法是在add里面调用的。

![941de1a1ce21f6bc602c2849a9837186.png]({{site.baseurl}}/img/941de1a1ce21f6bc602c2849a9837186.png)


再看grow方法

![6df080176de72c4525c60f0254062163.png]({{site.baseurl}}/img/6df080176de72c4525c60f0254062163.png)


跟进到Arrays这个工具类，很简单

![41bf227166e377c39423fd20c18b4aef.png]({{site.baseurl}}/img/41bf227166e377c39423fd20c18b4aef.png)


再看copyOf()方法

![7ff4b2c32138515cf7859fa586073e72.png]({{site.baseurl}}/img/7ff4b2c32138515cf7859fa586073e72.png)


最后我们来看一下System.arraycopy()方法，好奇怪，这个方法只有定义，却没有实现，方法用了一个native来修饰。native的方法，是由其它语言来实现的，一般是(C或C++)，所以这儿没有实现代码。这是一个数组拷贝方法，大家还在写for循环拷贝数组吗？以后多用这个方法吧，简单又方便还能获得得更好的性能。

![b7f7def7b4b142065641d9bf306e4754.png]({{site.baseurl}}/img/b7f7def7b4b142065641d9bf306e4754.png)


注：native方法，我们会后续会讲解，我们先关注本章内容。
由于数组内容目前为空，相当于没有拷贝。折腾了这么久，原来只是为了创建一个默认长度为10的Object[]数组，有些朋友说，直接new不就行了，这么费劲，其实这里面大有文章，别急，稍后会说，继续画图。

![6380161cd325280b6b1217cb16165248.png]({{site.baseurl}}/img/6380161cd325280b6b1217cb16165248.png)

再回过头来看，add()这个方法，继续往下执行：

![2270e5b68da068f6f3b6d5a706c8f8d9.png]({{site.baseurl}}/img/2270e5b68da068f6f3b6d5a706c8f8d9.png)


很简单，size现在是0，就是把传进来的这个e(这里是person1)，放到list1的elementData[]下标为0的数组里面，同时size加1，老规矩，上图。

![2f45939b7b04ee2f74f648406828e006.png]({{site.baseurl}}/img/2f45939b7b04ee2f74f648406828e006.png)

注意看红框里，虽然我们list1里的elementData数组的长度是10，但是size是1，size是逻辑长度，并不是数组长度。

现在debug一下，验证我们图里的内容：

![ad8bdcb222a1cfbf13d06099d54ade60.png]({{site.baseurl}}/img/ad8bdcb222a1cfbf13d06099d54ade60.png)


好的，执行一下本文开始那段代码，看结果：

![e9b24139e20218bb53492e29fb0d812c.png]({{site.baseurl}}/img/e9b24139e20218bb53492e29fb0d812c.png)


顺便看一看size()方法的源码：

![d7e0cc1c5e0bc1dda384f87bdfa9cfbf.png]({{site.baseurl}}/img/d7e0cc1c5e0bc1dda384f87bdfa9cfbf.png)


有人说，呀，就一个元素，在堆内存中占了10个位置，好浪费呀，没办法，你要享受ArrayList的便利与丰富的API，就得牺牲一下空间作为代价。
