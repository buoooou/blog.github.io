---
layout: post
published: true
title: AbstractFactory抽象工厂模式
---
# AbstractFactory抽象工厂模式

## 抽象工厂模式的概念

　　抽象工厂模式是所有形态的工厂模式中最为抽象和最具一般性的一种形态。抽象工厂模式是指当有多个抽象角色时，使用的一种工厂模式。
  
## 应用场景　　
　　
    第一种情况是对于某个产品，调用者清楚地知道应该使用哪个具体工厂服务，实例化该具体工厂，生产出具体的产品来。Java Collection中的iterator() 方法即属于这种情况。
　　
	第二种情况，只是需要一种产品，而不想知道也不需要知道究竟是哪个工厂为生产的，即最终选用哪个具体工厂的决定权在生产者一方，它们根据当前系统的情况来实例化一个具体的工厂返回给使用者，而这个决策过程这对于使用者来说是透明的。
  
## DEMO

    /**
     * 产品的抽象接口  人类
     * @author liaowp
     *
     */
    public interface Human {

        public void say();

    }
    
    /**
     * man  男人
     * @author liaowp
     *
     */
    public class Man implements Human {

        /* say method
         * @see com.roc.factory.Human#say()
         */
        @Override
        public void say() {
            System.out.println("男人");
        }

    }

	/**女人
     * @author liaowp
     *
     */
    public class Woman implements Human {

        /* say method
         * @see com.roc.factory.Human#say()
         */
        @Override
        public void say() {
            System.out.println("女人");
        }

    }
    
    /**
     * 工厂接口类
     * @author liaowp
     *
     */
    public interface Factory {

        public Human crateMan();

    }
    
    /**
     * 创造男人工厂类
     * @author liaowp
     *
     */
    public class ManFactory implements Factory{

        public Human crateMan() {
            return new Man();
        }

    }

    /**
     * 创造女人工厂类
     * @author liaowp
     *
     */
    public class WomanFactory implements Factory{

        @Override
        public Human crateMan() {
            // TODO Auto-generated method stub
            return new Woman();
        }

    }
    
    ／**
     * 抽象工厂测试
     * @author liaowp
     *
     */
    public class Client {
        public static void main(String[] args) {    
            Factory factory=new ManFactory();
            Human  man2=factory.crateMan();
            man2.say();

        }
    }
    
    