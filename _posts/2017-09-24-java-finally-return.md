---
layout: post
published: true
title: java finally块和return
---
## finally块和return

首先一个不容易理解的事实：在 try块中即便有return，break，continue等改变执行流的语句，finally也会执行。

    public static void main(String[] args)
    {
        int re = bar();
        System.out.println(re);
    }
    private static int bar() 
    {
        try{
            return 5;
        } finally{
            System.out.println("finally");
        }
    }
    /*输出：
    finally
    */

很多人面对这个问题时，总是在归纳执行的顺序和规律，不过我觉得还是很难理解。我自己总结了一个方法。用如下GIF图说明。

￼![hevc.gif.gif]({{site.baseurl}}/img/hevc.gif.gif)


也就是说：try…catch…finally中的return 只要能执行，就都执行了，他们共同向同一个内存地址（假设地址是0×80）写入返回值，后执行的将覆盖先执行的数据，而真正被调用者取的返回值就是最后一次写入的。那么，按照这个思想，下面的这个例子也就不难理解了。

finally中的return 会覆盖 try 或者catch中的返回值。

    public static void main(String[] args)
        {
            int result;

            result  =  foo();
            System.out.println(result);     /////////2

            result = bar();
            System.out.println(result);    /////////2
        }

        @SuppressWarnings("finally")
        public static int foo()
        {
            trz{
                int a = 5 / 0;
            } catch (Exception e){
                return 1;
            } finally{
                return 2;
            }

        }

        @SuppressWarnings("finally")
        public static int bar()
        {
            try {
                return 1;
            }finally {
                return 2;
            }
        }

finally中的return会抑制（消灭）前面try或者catch块中的异常

    class TestException
    {
        public static void main(String[] args)
        {
            int result;
            try{
                result = foo();
                System.out.println(result);           //输出100
            } catch (Exception e){
                System.out.println(e.getMessage());    //没有捕获到异常
            }


            try{
                result  = bar();
                System.out.println(result);           //输出100
            } catch (Exception e){
                System.out.println(e.getMessage());    //没有捕获到异常
            }
        }

        //catch中的异常被抑制
        @SuppressWarnings("finally")
        public static int foo() throws Exception
        {
            try {
                int a = 5/0;
                return 1;
            }catch(ArithmeticException amExp) {
                throw new Exception("我将被忽略，因为下面的finally中使用了return");
            }finally {
                return 100;
            }
        }

        //try中的异常被抑制
        @SuppressWarnings("finally")
        public static int bar() throws Exception
        {
            try {
                int a = 5/0;
                return 1;
            }finally {
                return 100;
            }
        }
    }

finally中的异常会覆盖（消灭）前面try或者catch中的异常

    class TestException
    {
        public static void main(String[] args)
        {
            int result;
            try{
                result = foo();
            } catch (Exception e){
                System.out.println(e.getMessage());    //输出：我是finaly中的Exception
            }


            try{
                result  = bar();
            } catch (Exception e){
                System.out.println(e.getMessage());    //输出：我是finaly中的Exception
            }
        }

        //catch中的异常被抑制
        @SuppressWarnings("finally")
        public static int foo() throws Exception
        {
            try {
                int a = 5/0;
                return 1;
            }catch(ArithmeticException amExp) {
                throw new Exception("我将被忽略，因为下面的finally中抛出了新的异常");
            }finally {
                throw new Exception("我是finaly中的Exception");
            }
        }

        //try中的异常被抑制
        @SuppressWarnings("finally")
        public static int bar() throws Exception
        {
            try {
                int a = 5/0;
                return 1;
            }finally {
                throw new Exception("我是finaly中的Exception");
            }

        }
    }

上面的3个例子都异于常人的编码思维，因此我建议：

* 不要在fianlly中使用return。
* 不要在finally中抛出异常。
* 减轻finally的任务，不要在finally中做一些其它的事情，finally块仅仅用来释放资源是最合适的。
* 将尽量将所有的return写在函数的最后面，而不是try … catch … finally中。

