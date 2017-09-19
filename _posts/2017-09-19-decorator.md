---
layout: post
published: true
title: Decorator装饰者模式
---
# Decorator装饰者模式

## 意图

动态地给一个对象添加一些额外的职责。就增加功能来说， Decorator模式相比生成子类更为灵活。该模式以对客 户端透明的方式扩展对象的功能。

## 适用环境

（1）在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责。

（2）处理那些可以撤消的职责。

（3）当不能采用生成子类的方法进行扩充时。一种情况是，可能有大量独立的扩展，为支持每一种组合将产生大量的 子类，使得子类数目呈爆炸性增长。另一种情况可能是因为类定义被隐藏，或类定义不能用于生成子类。

## 参与者

1.Component（被装饰对象的基类）

      定义一个对象接口，可以给这些对象动态地添加职责。

2.ConcreteComponent（具体被装饰对象）

      定义一个对象，可以给这个对象添加一些职责。

3.Decorator（装饰者抽象类）

      维持一个指向Component实例的引用，并定义一个与Component接口一致的接口。

4.ConcreteDecorator（具体装饰者）

      具体的装饰对象，给内部持有的具体被装饰对象，增加具体的职责。

## 涉及角色

（1）抽象组件:定义一个抽象接口，来规范准备附加功能的类

（2）具体组件：将要被附加功能的类，实现抽象构件角色接口

（3）抽象装饰者：持有对具体构件角色的引用并定义与抽象构件角色一致的接口

（4）具体装饰：实现抽象装饰者角色，负责对具体构件添加额外功能。

## DEMO代码 

Component 

    public interface Person {

        void eat();
    }
 

ConcreteComponent 

    public class Man implements Person {

        public void eat() {
            System.out.println("男人在吃");
        }
    }

Decorator

    public abstract class Decorator implements Person {

        protected Person person;

        public void setPerson(Person person) {
            this.person = person;
        }

        public void eat() {
            person.eat();
        }
	}

ConcreteDectrator

    public class ManDecoratorA extends Decorator {

        public void eat() {
            super.eat();
            reEat();
            System.out.println("ManDecoratorA类");
        }

        public void reEat() {
            System.out.println("再吃一顿饭");
        }
    }
    public class ManDecoratorB extends Decorator {

        public void eat() {
            super.eat();
            System.out.println("===============");
            System.out.println("ManDecoratorB类");
        }
    }

Test 

    public class Test {

        public static void main(String[] args) {
            Man man = new Man();
            ManDecoratorA md1 = new ManDecoratorA();
            ManDecoratorB md2 = new ManDecoratorB();

            md1.setPerson(man);
            md2.setPerson(md1);
            md2.eat();
        }
    }

## 装饰者模式小结：

OO原则：动态地将责任附加到对象上。想要扩展功能， 装饰者提供有别于继承的另一种选择。

## 要点：

1、继承属于扩展形式之一，但不见得是达到弹性设计的最佳方案。
2、在我们的设计中，应该允许行为可以被扩展，而不须修改现有的代码。
3、组合和委托可用于在运行时动态地加上新的行为。
4、除了继承，装饰者模式也可以让我们扩展行为。
5、装饰者模式意味着一群装饰者类， 这些类用来包装具体组件。
6、装饰者类反映出被装饰的组件类型（实际上，他们具有相同的类型，都经过接口或继承实现）。
7、装饰者可以在被装饰者的行为前面与/或后面加上自己的行为，甚至将被装饰者的行为整个取代掉，而达到特定的目的。
8、你可以有无数个装饰者包装一个组件。
9、 装饰者一般对组建的客户是透明的，除非客户程序依赖于组件的具体类型。