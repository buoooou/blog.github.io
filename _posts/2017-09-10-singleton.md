---
layout: post
published: true
title: Singleton单例模式
---
# Singleton单例模式

## 解法一：只适合单线程环境（不好）

    /**
     * @author xiaoping
     *
     */
    public class Singleton {
        private static Singleton instance=null;
        private Singleton(){

        }
        public static Singleton getInstance(){
            if(instance==null){
                instance=new Singleton();
            }
            return instance;
        }
    }
    
注解:Singleton的静态属性instance中，只有instance为null的时候才创建一个实例，构造函数私有，确保每次都只创建一个，避免重复创建。
缺点：只在单线程的情况下正常运行，在多线程的情况下，就会出问题。例如：当两个线程同时运行到判断instance是否为空的if语句，并且instance确实没有创建好时，那么两个线程都会创建一个实例。

## 解法二：多线程的情况可以用。（懒汉式，不好）

    public class Singleton {
        private static Singleton instance=null;
        private Singleton(){

        }
        public static synchronized Singleton getInstance(){
            if(instance==null){
                instance=new Singleton();
            }
            return instance;
        }
    }
    
注解：在解法一的基础上加上了同步锁，使得在多线程的情况下可以用。例如：当两个线程同时想创建实例，由于在一个时刻只有一个线程能得到同步锁，当第一个线程加上锁以后，第二个线程只能等待。第一个线程发现实例没有创建，创建之。第一个线程释放同步锁，第二个线程才可以加上同步锁，执行下面的代码。由于第一个线程已经创建了实例，所以第二个线程不需要创建实例。保证在多线程的环境下也只有一个实例。
缺点：每次通过getInstance方法得到singleton实例的时候都有一个试图去获取同步锁的过程。而众所周知，加锁是很耗时的。能避免则避免。

## 解法三：加同步锁时，前后两次判断实例是否存在（可行）

    public class Singleton {
        private static Singleton instance=null;
        private Singleton(){

        }
        public static Singleton getInstance(){
            if(instance==null){
                synchronized(Singleton.class){
                    if(instance==null){
                        instance=new Singleton();
                    }
                }
            }
            return instance;
        }
    }
    
注解：只有当instance为null时，需要获取同步锁，创建一次实例。当实例被创建，则无需试图加锁。
缺点：用双重if判断，复杂，容易出错。

这里要提到Java中的指令重排优化。所谓指令重排优化是指在不改变原语义的情况下，通过调整指令的执行顺序让程序运行的更快。JVM中并没有规定编译器优化相关的内容，也就是说JVM可以自由的进行指令重排序的优化。

这个问题的关键就在于由于指令重排优化的存在，导致初始化Singleton和将对象地址赋给instance字段的顺序是不确定的。在某个线程创建单例对象时，在构造方法被调用之前，就为该对象分配了内存空间并将对象的字段设置为默认值。此时就可以将分配的内存地址赋值给instance字段了，然而该对象可能还没有初始化。若紧接着另外一个线程来调用getInstance，取到的就是状态不正确的对象，程序就会出错。

以上就是双重校验锁会失效的原因，不过还好在JDK1.5及之后版本增加了volatile关键字。volatile的一个语义是禁止指令重排序优化，也就保证了instance变量被赋值的时候对象已经是初始化过的，从而避免了上面说到的问题。代码如下：
     
     public class Singleton {  
        private static volatile Singleton instance = null;  
        private Singleton(){}  
        public static Singleton getInstance() {  
            if (instance == null) {  
                synchronized (Singleton.class) {  
                    if (instance == null) {  
                        instance = new Singleton();  
                    }  
                }  
            }  
            return instance;  
        }  
    }  

## 解法四：饿汉式（建议使用）

    public class Singleton {
        private static Singleton instance=new Singleton();
        private Singleton(){

        }
        public static Singleton getInstance(){
            return instance;
        }
    }
    
注解：初试化静态的instance创建一次。如果我们在Singleton类里面写一个静态的方法不需要创建实例，它仍然会早早的创建一次实例。而降低内存的使用率。

缺点：没有lazy loading的效果，从而降低内存的使用率。

## 解法五：静态内部内。（建议使用）

    public class Singleton {
        private Singleton(){

        }
        private static class SingletonHolder{
            private final static Singleton instance=new Singleton();
        }
        public static Singleton getInstance(){
            return SingletonHolder.instance;
        }
    }
    
注解：定义一个私有的内部类，在第一次用这个嵌套类时，会创建一个实例。而类型为SingletonHolder的类，只有在Singleton.getInstance()中调用，由于私有的属性，他人无法使用SingleHolder，不调用Singleton.getInstance()就不会创建实例。
优点：达到了lazy loading的效果，即按需创建实例。 

## 总结
   
   本文总结了五种Java中实现单例的方法，其中前两种都不够完美，双重校验锁和静态内部类的方式可以解决大部分问题，平时工作中使用的最多的也是这两种方式。枚举方式虽然很完美的解决了各种问题，但是这种写法多少让人感觉有些生疏。个人的建议是，在没有特殊需求的情况下，使用第三种和第四种方式实现单例模式。