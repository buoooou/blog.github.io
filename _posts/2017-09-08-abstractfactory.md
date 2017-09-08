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

    //抽象工厂角色
    public interface AbstractFactory{
      public ProductA createProductA();
      public ProductB createProductB();
    }

    //抽象产品类A
    public interface AbstractProductA
    {
    }

    //抽象产品类B
    public interface AbstractProductB
    {
    }

    //具体产品类ProdcutA1
    public class ProductA1 implements AbstractProductA 
    {
      public ProductA1()
      {
      }
    }

    //具体产品类ProdcutA2
    public class ProductA2 implements AbstractProductA
    {
      public ProductA2()
      {
      }
    }

    //具体产品类ProductB1
    public class ProductB1 implements AbstractProductB
    {
      public ProductB1()
      {
      }
    } 

    //具体产品类ProductB2
    public class ProductB2 implements AbstractProductB
    {
      public ProductB2()
      {
      }
    }

    //具体工厂类1
    public class ConcreteFactory1 implements AbstractFactory{
      public AbstractProductA createProductA(){
      return new ProductA1();
      }
      public AbstractProductB createProductB(){
      return new ProductB1();
        }
    }

    //具体工厂类2
    public class ConcreteFactory2 implements Creator{
      public AbstractProductA createProductA(){
      return new ProductA2();
      }
      public AbstractProductB createProductB(){
      return new ProductB2();
      }
    } 
