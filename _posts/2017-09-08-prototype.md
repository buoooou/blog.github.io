---
layout: post
published: true
title: Prototype原型模式
---
# Prototype原型模式

原型模式属于对象的创建模式。通过给出一个原型对象来指明所有创建的对象的类型，然后用复制这个原型对象的办法创建出更多同类型的对象。这就是选型模式的用意。

原型模式有两种表现形式：（1）简单形式、（2）登记形式，这两种表现形式仅仅是原型模式的不同实现。

## 简单形式的原型模式

这种形式涉及到三个角色：

　　（1）客户(Client)角色：客户类提出创建对象的请求。

　　（2）抽象原型(Prototype)角色：这是一个抽象角色，通常由一个Java接口或Java抽象类实现。此角色给出所有的具体原型类所需的接口。

　　（3）具体原型（Concrete Prototype）角色：被复制的对象。此角色需要实现抽象的原型角色所要求的接口。

### DEMO

抽象原型角色

    public interface Prototype{
        /**
         * 克隆自身的方法
         * @return 一个从自身克隆出来的对象
         */
        public Object clone();
    }
    
具体原型角色

    public class ConcretePrototype1 implements Prototype {
        public Prototype clone(){
            //最简单的克隆，新建一个自身对象，由于没有属性就不再复制值了
            Prototype prototype = new ConcretePrototype1();
            return prototype;
        }
    }
    
    public class ConcretePrototype2 implements Prototype {
        public Prototype clone(){
            //最简单的克隆，新建一个自身对象，由于没有属性就不再复制值了
            Prototype prototype = new ConcretePrototype2();
            return prototype;
        }
    }
    
客户端角色

    public class Client {
        /**
         * 持有需要使用的原型接口对象
         */
        private Prototype prototype;
        /**
         * 构造方法，传入需要使用的原型接口对象
         */
        public Client(Prototype prototype){
            this.prototype = prototype;
        }
        public void operation(Prototype example){
            //需要创建原型接口的对象
            Prototype copyPrototype = prototype.clone();

        }
    }
    
## 登记形式的原型模式

作为原型模式的第二种形式，它多了一个原型管理器(PrototypeManager)角色，该角色的作用是：创建具体原型类的对象，并记录每一个被创建的对象。

### DEMO

抽象原型角色

    public interface Prototype{
        public Prototype clone();
        public String getName();
        public void setName(String name);
    }
    
具体原型角色

    public class ConcretePrototype1 implements Prototype {
        private String name;
        public Prototype clone(){
            ConcretePrototype1 prototype = new ConcretePrototype1();
            prototype.setName(this.name);
            return prototype;
        }
        public String toString(){
            return "Now in Prototype1 , name = " + this.name;
        }
        @Override
        public String getName() {
            return name;
        }

        @Override
        public void setName(String name) {
            this.name = name;
        }
	}
    
    public class ConcretePrototype2 implements Prototype {
        private String name;
        public Prototype clone(){
            ConcretePrototype2 prototype = new ConcretePrototype2();
            prototype.setName(this.name);
            return prototype;
        }
        public String toString(){
            return "Now in Prototype2 , name = " + this.name;
        }
        @Override
        public String getName() {
            return name;
        }

        @Override
        public void setName(String name) {
            this.name = name;
        }
    }
    
原型管理器角色保持一个聚集，作为对所有原型对象的登记，这个角色提供必要的方法，供外界增加新的原型对象和取得已经登记过的原型对象。

    public class PrototypeManager {
        /**
         * 用来记录原型的编号和原型实例的对应关系
         */
        private static Map<String,Prototype> map = new HashMap<String,Prototype>();
        /**
         * 私有化构造方法，避免外部创建实例
         */
        private PrototypeManager(){}
        /**
         * 向原型管理器里面添加或是修改某个原型注册
         * @param prototypeId 原型编号
         * @param prototype    原型实例
         */
        public synchronized static void setPrototype(String prototypeId , Prototype prototype){
            map.put(prototypeId, prototype);
        }
        /**
         * 从原型管理器里面删除某个原型注册
         * @param prototypeId 原型编号
         */
        public synchronized static void removePrototype(String prototypeId){
            map.remove(prototypeId);
        }
        /**
         * 获取某个原型编号对应的原型实例
         * @param prototypeId    原型编号
         * @return    原型编号对应的原型实例
         * @throws Exception    如果原型编号对应的实例不存在，则抛出异常
         */
        public synchronized static Prototype getPrototype(String prototypeId) throws Exception{
            Prototype prototype = map.get(prototypeId);
            if(prototype == null){
                throw new Exception("您希望获取的原型还没有注册或已被销毁");
            }
            return prototype;
        }
    }
    
客户端角色

    public class Client {
        public static void main(String[]args){
            try{
                Prototype p1 = new ConcretePrototype1();
                PrototypeManager.setPrototype("p1", p1);
                //获取原型来创建对象
                Prototype p3 = PrototypeManager.getPrototype("p1").clone();
                p3.setName("张三");
                System.out.println("第一个实例：" + p3);
                //有人动态的切换了实现
                Prototype p2 = new ConcretePrototype2();
                PrototypeManager.setPrototype("p1", p2);
                //重新获取原型来创建对象
                Prototype p4 = PrototypeManager.getPrototype("p1").clone();
                p4.setName("李四");
                System.out.println("第二个实例：" + p4);
                //有人注销了这个原型
                PrototypeManager.removePrototype("p1");
                //再次获取原型来创建对象
                Prototype p5 = PrototypeManager.getPrototype("p1").clone();
                p5.setName("王五");
                System.out.println("第三个实例：" + p5);
            }catch(Exception e){
                e.printStackTrace();
            }
        }
    }
       
## 两种形式的比较

　　简单形式和登记形式的原型模式各有其长处和短处。

　　如果需要创建的原型对象数目较少而且比较固定的话，可以采取第一种形式。在这种情况下，原型对象的引用可以由客户端自己保存。

　　如果要创建的原型对象数目不固定的话，可以采取第二种形式。在这种情况下，客户端不保存对原型对象的引用，这个任务被交给管理员对象。在复制一个原型对象之前，客户端可以查看管理员对象是否已经有一个满足要求的原型对象。如果有，可以直接从管理员类取得这个对象引用；如果没有，客户端就需要自行复制此原型对象。
  
## 原型模式的优点

　　原型模式允许在运行时动态改变具体的实现类型。原型模式可以在运行期间，由客户来注册符合原型接口的实现类型，也可以动态地改变具体的实现类型，看起来接口没有任何变化，但其实运行的已经是另外一个类实例了。因为克隆一个原型就类似于实例化一个类。

## 原型模式的缺点

　　原型模式最主要的缺点是每一个类都必须配备一个克隆方法。配备克隆方法需要对类的功能进行通盘考虑，这对于全新的类来说不是很难，而对于已经有的类不一定很容易，特别是当一个类引用不支持序列化的间接对象，或者引用含有循环结构的时候。