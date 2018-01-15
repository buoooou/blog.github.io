---
layout: post
published: true
title: 扩展 ThreadPoolExecutor 的一种办法
---
# 扩展 ThreadPoolExecutor 的一种办法

## 概述

在JAVA的世界里,如果想并行的执行一些任务,可以使用ThreadPoolExecutor。
大部分情况下直接使用ThreadPoolExecutor就可以满足要求了,但是在某些场景下,比如瞬时大流量的,为了提高响应和吞吐量,最好还是扩展一下ThreadPoolExecutor。

全宇宙的JAVA IT人士应该都知道ThreadPoolExecutor的执行流程：

- core线程还能应付的,则不断的创建新的线程;
- core线程无法应付,则将任务扔到队列里面;
- 队列满了(意味着插入任务失败),则开始创建MAX线程,线程数达到MAX后,队列还一直是满的,则抛出RejectedExecutionException.


这个执行流程有个小问题,就是当core线程无法应付请求的时候,会立刻将任务添加到队列中,如果队列非常长,而任务又非常多,那么将会有频繁的任务入队列和任务出队列的操作。

根据实际的压测发现,这种操作也是有一定消耗的。其实JAVA提供的SynchronousQueue队列是一个零长度的队列,任务都是直接由生产者递交给消费者,中间没有入队列的过程,可见JAVA API的设计者也是有考虑过入队列这种操作的开销。

另外，任务一多,立刻扔到队列里,而MAX线程又不干活,如果队列里面太多任务了,只有可怜的core线程在忙,也是会影响性能的。

当core线程无法应付请求的时候,能不能延后入队列这个操作呢? 让MAX线程尽快启动起来,帮忙处理任务。

也即是说,当core线程无法应付请求的时候,如果当前线程池中的线程数量还小于MAX线程数的时候,继续创建新的线程处理任务,一直到线程数量到达MAX后,才将任务插入到队列里。

我们通过覆盖队列的offer方法来实现这个目标。

      @Override
    public  boolean offer(Runnable o) {
        int currentPoolThreadSize = executor.getPoolSize();
        //如果线程池里的线程数量已经到达最大,将任务添加到队列中
        if (currentPoolThreadSize == executor.getMaximumPoolSize()) {
            return super.offer(o);
        }
        //说明有空闲的线程,这个时候无需创建core线程之外的线程,而是把任务直接丢到队列里即可
        if (executor.getSubmittedTaskCount() < currentPoolThreadSize) {
            return super.offer(o);
        }

        //如果线程池里的线程数量还没有到达最大,直接创建线程,而不是把任务丢到队列里面
        if (currentPoolThreadSize < executor.getMaximumPoolSize()) {
            return false;
        }

        return super.offer(o);
    }

注意其中的

    if (executor.getSubmittedTaskCount() < currentPoolThreadSize) {
            return super.offer(o);
    }

是表示core线程仍然能处理的来,同时又有空闲线程的情况,将任务插入到队列中。 如何判断线程池中有空闲线程呢？ 可以使用一个计数器来实现,每当execute方法被执行的时候,计算器加1,当afterExecute被执行后,计数器减1.

    @Override
    public void execute(Runnable command) {
        submittedTaskCount.incrementAndGet();
        //代码未完整,待补充。。。。。
    }

    @Override
       protected void afterExecute(Runnable r, Throwable t) {
           submittedTaskCount.decrementAndGet();
       }

这样,当

	executor.getSubmittedTaskCount() < currentPoolThreadSize

的时候,说明有空闲线程。

## 完整代码

EnhancedThreadPoolExecutor类

    package executer;

    import java.util.concurrent.*;
    import java.util.concurrent.atomic.AtomicInteger;

    public class EnhancedThreadPoolExecutor extends java.util.concurrent.ThreadPoolExecutor {

        /**
         * 计数器,用于表示已经提交到队列里面的task的数量,这里task特指还未完成的task。
         * 当task执行完后,submittedTaskCount会减1的。
         */
        private final AtomicInteger submittedTaskCount = new AtomicInteger(0);

        public EnhancedThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, TaskQueue workQueue) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, new ThreadPoolExecutor.AbortPolicy());
            workQueue.setExecutor(this);
        }

        /**
         * 覆盖父类的afterExecute方法,当task执行完成后,将计数器减1
         */
        @Override
        protected void afterExecute(Runnable r, Throwable t) {
            submittedTaskCount.decrementAndGet();
        }


        public int getSubmittedTaskCount() {
            return submittedTaskCount.get();
        }


        /**
         * 覆盖父类的execute方法,在任务开始执行之前,计数器加1。
         */
        @Override
        public void execute(Runnable command) {
            submittedTaskCount.incrementAndGet();
            try {
                super.execute(command);
            } catch (RejectedExecutionException rx) {
                //当发生RejectedExecutionException,尝试再次将task丢到队列里面,如果还是发生RejectedExecutionException,则直接抛出异常。
                BlockingQueue<Runnable> taskQueue = super.getQueue();
                if (taskQueue instanceof TaskQueue) {
                    final TaskQueue queue = (TaskQueue)taskQueue;
                    if (!queue.forceTaskIntoQueue(command)) {
                        submittedTaskCount.decrementAndGet();
                        throw new RejectedExecutionException("队列已满");
                    }
                } else {
                    submittedTaskCount.decrementAndGet();
                    throw rx;
                }
            }
        }
    }

TaskQueue

    package executer;

    import java.util.concurrent.LinkedBlockingQueue;
    import java.util.concurrent.RejectedExecutionException;

    public class TaskQueue extends LinkedBlockingQueue<Runnable> {
        private EnhancedThreadPoolExecutor executor;

        public TaskQueue(int capacity) {
            super(capacity);
        }

        public void setExecutor(EnhancedThreadPoolExecutor exec) {
            executor = exec;
        }

        public boolean forceTaskIntoQueue(Runnable o) {
            if (executor.isShutdown()) {
                throw new RejectedExecutionException("Executor已经关闭了,不能将task添加到队列里面");
            }
            return super.offer(o);
        }

        @Override
        public  boolean offer(Runnable o) {
            int currentPoolThreadSize = executor.getPoolSize();
            //如果线程池里的线程数量已经到达最大,将任务添加到队列中
            if (currentPoolThreadSize == executor.getMaximumPoolSize()) {
                return super.offer(o);
            }
            //说明有空闲的线程,这个时候无需创建core线程之外的线程,而是把任务直接丢到队列里即可
            if (executor.getSubmittedTaskCount() < currentPoolThreadSize) {
                return super.offer(o);
            }

            //如果线程池里的线程数量还没有到达最大,直接创建线程,而不是把任务丢到队列里面
            if (currentPoolThreadSize < executor.getMaximumPoolSize()) {
                return false;
            }

            return super.offer(o);
        }
    }

TestExecuter

    package executer;

    import java.util.concurrent.TimeUnit;

    public class TestExecuter {
        private static final int CORE_SIZE = 5;

        private static final int MAX_SIZE = 10;

        private static final long KEEP_ALIVE_TIME = 30;

        private static final int QUEUE_SIZE = 5;

        static EnhancedThreadPoolExecutor executor = new EnhancedThreadPoolExecutor(CORE_SIZE,MAX_SIZE,KEEP_ALIVE_TIME, TimeUnit.SECONDS , new TaskQueue(QUEUE_SIZE));

        public static void main(String[] args){
            for (int i = 0; i < 15; i++) {
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            Thread.currentThread().sleep(1000);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                });

                System.out.println("线程池中现在的线程数目是："+executor.getPoolSize()+",  队列中正在等待执行的任务数量为："+ executor.getQueue().size());
            }
        }
    }

先运行一下代码,看看是否如何预期。直接执行TestExecuter类中的main方法,运行结果如下:

线程池中现在的线程数目是：1,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：2,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：3,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：4,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：5,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：6,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：7,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：8,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：9,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：10,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：10,  队列中正在等待执行的任务数量为：1
线程池中现在的线程数目是：10,  队列中正在等待执行的任务数量为：2
线程池中现在的线程数目是：10,  队列中正在等待执行的任务数量为：3
线程池中现在的线程数目是：10,  队列中正在等待执行的任务数量为：4
线程池中现在的线程数目是：10,  队列中正在等待执行的任务数量为：5

可以看到当线程数增加到core数量的时候,队列中是没有任务的。一直到线程数量增加到MAX数量,也即是10的时候,队列中才开始有任务。符合我们的预期。

如果我们注释掉TaskQueue类中的offer方法,也即是不覆盖队列的offer方法,那么运行结果如下:

线程池中现在的线程数目是：1,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：2,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：3,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：4,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：5,  队列中正在等待执行的任务数量为：0
线程池中现在的线程数目是：5,  队列中正在等待执行的任务数量为：1
线程池中现在的线程数目是：5,  队列中正在等待执行的任务数量为：2
线程池中现在的线程数目是：5,  队列中正在等待执行的任务数量为：3
线程池中现在的线程数目是：5,  队列中正在等待执行的任务数量为：4
线程池中现在的线程数目是：5,  队列中正在等待执行的任务数量为：5
线程池中现在的线程数目是：6,  队列中正在等待执行的任务数量为：5
线程池中现在的线程数目是：7,  队列中正在等待执行的任务数量为：5
线程池中现在的线程数目是：8,  队列中正在等待执行的任务数量为：5
线程池中现在的线程数目是：9,  队列中正在等待执行的任务数量为：5
线程池中现在的线程数目是：10,  队列中正在等待执行的任务数量为：5

可以看到当线程数增加到core数量的时候,队列中已经有任务了。

## 进一步思考

在使用ThreadPoolExecutor的时候,如果发生了RejectedExecutionException,该如何处理？本文中的代码是采用了重新将任务尝试插入到队列中,如果还是失败则直接将reject异常抛出去。

    @Override
        public void execute(Runnable command) {
            submittedTaskCount.incrementAndGet();
            try {
                super.execute(command);
            } catch (RejectedExecutionException rx) {
                //当发生RejectedExecutionException,尝试再次将task丢到队列里面,如果还是发生RejectedExecutionException,则直接抛出异常。
                BlockingQueue<Runnable> taskQueue = super.getQueue();
                if (taskQueue instanceof TaskQueue) {
                    final TaskQueue queue = (TaskQueue)taskQueue;
                    if (!queue.forceTaskIntoQueue(command)) {
                        submittedTaskCount.decrementAndGet();
                        throw new RejectedExecutionException("队列已满");
                    }
                } else {
                    submittedTaskCount.decrementAndGet();
                    throw rx;
                }
            }
        }

TaskQueue类提供了forceTaskIntoQueue方法,将任务插入到队列中。

还有另一种解决方案,就是使用另外一个线程池来执行任务,当第一个线程池抛出Reject异常时,catch住它,并使用第二个线程池处理任务。
