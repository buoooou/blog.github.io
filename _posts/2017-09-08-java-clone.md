---
layout: post
published: true
title: Java Clone克隆方法
---
# java克隆方法

Java的所有类都是从java.lang.Object类继承而来的，而Object类提供protected Object clone()方法对对象进行复制，子类当然也可以把这个方法置换掉，提供满足自己需要的复制方法。对象的复制有一个基本问题，就是对象通常都有对其他的对象的引用。当使用Object类的clone()方法来复制一个对象时，此对象对其他对象的引用也同时会被复制一份

　　Java语言提供的Cloneable接口只起一个作用，就是在运行时期通知Java虚拟机可以安全地在这个类上使用clone()方法。通过调用这个clone()方法可以得到一个对象的复制。由于Object类本身并不实现Cloneable接口，因此如果所考虑的类没有实现Cloneable接口时，调用clone()方法会抛出CloneNotSupportedException异常。
	
## 克隆满足的条件

　　clone()方法将对象复制了一份并返还给调用者。所谓“复制”的含义与clone()方法是怎么实现的。一般而言，clone()方法满足以下的描述：

　　（1）对任何的对象x，都有：x.clone()!=x。换言之，克隆对象与原对象不是同一个对象。

　　（2）对任何的对象x，都有：x.clone().getClass() == x.getClass()，换言之，克隆对象与原对象的类型一样。

　　（3）如果对象x的equals()方法定义其恰当的话，那么x.clone().equals(x)应当成立的。

　　在JAVA语言的API中，凡是提供了clone()方法的类，都满足上面的这些条件。JAVA语言的设计师在设计自己的clone()方法时，也应当遵守着三个条件。一般来说，上面的三个条件中的前两个是必需的，而第三个是可选的。

## 浅克隆和深克隆

　　无论你是自己实现克隆方法，还是采用Java提供的克隆方法，都存在一个浅度克隆和深度克隆的问题。

　　**浅度克隆**
  
　　只负责克隆按值传递的数据（比如基本数据类型、String类型），而不复制它所引用的对象，换言之，所有的对其他对象的引用都仍然指向原来的对象。

　　**深度克隆**
  
　　除了浅度克隆要克隆的值外，还负责克隆引用类型的数据。那些引用其他对象的变量将指向被复制过的新对象，而不再是原有的那些被引用的对象。换言之，深度克隆把要复制的对象所引用的对象都复制了一遍，而这种对被引用到的对象的复制叫做间接复制。

　　深度克隆要深入到多少层，是一个不易确定的问题。在决定以深度克隆的方式复制一个对象的时候，必须决定对间接复制的对象时采取浅度克隆还是继续采用深度克隆。因此，在采取深度克隆时，需要决定多深才算深。此外，在深度克隆的过程中，很可能会出现循环引用的问题，必须小心处理。

## 利用序列化实现深度克隆

　　把对象写到流里的过程是序列化(Serialization)过程；而把对象从流中读出来的过程则叫反序列化(Deserialization)过程。应当指出的是，写到流里的是对象的一个拷贝，而原对象仍然存在于JVM里面。

　　在Java语言里深度克隆一个对象，常常可以先使对象实现Serializable接口，然后把对象（实际上只是对象的拷贝）写到一个流里（序列化），再从流里读回来（反序列化），便可以重建对象。

    public  Object deepClone() throws IOException, ClassNotFoundException{
        //将对象写到流里
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(this);
        //从流里读回来
        ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bis);
        return ois.readObject();
    }
    

　　这样做的前提就是对象以及对象内部所有引用到的对象都是可序列化的，否则，就需要仔细考察那些不可序列化的对象可否设成transient，从而将之排除在复制过程之外。

　　浅度克隆显然比深度克隆更容易实现，因为Java语言的所有类都会继承一个clone()方法，而这个clone()方法所做的正是浅度克隆。

　　有一些对象，比如线程(Thread)对象或Socket对象，是不能简单复制或共享的。不管是使用浅度克隆还是深度克隆，只要涉及这样的间接对象，就必须把间接对象设成transient而不予复制；或者由程序自行创建出相当的同种对象，权且当做复制件使用。
  
## DEMO

孙大圣本人用TheGreatestSage类代表

    public class TheGreatestSage {
        private Monkey monkey = new Monkey();

        public void change(){
            //克隆大圣本尊
            Monkey copyMonkey = (Monkey)monkey.clone();
            System.out.println("大圣本尊的生日是：" + monkey.getBirthDate());
            System.out.println("克隆的大圣的生日是：" + monkey.getBirthDate());
            System.out.println("大圣本尊跟克隆的大圣是否为同一个对象 " + (monkey == copyMonkey));
            System.out.println("大圣本尊持有的金箍棒 跟 克隆的大圣持有的金箍棒是否为同一个对象？ " + (monkey.getStaff() == copyMonkey.getStaff()));
        }

        public static void main(String[]args){
            TheGreatestSage sage = new TheGreatestSage();
            sage.change();
        }
    }

　　大圣本尊由Monkey类代表，这个类扮演具体原型角色：

    public class Monkey implements Cloneable {
        //身高
        private int height;
        //体重
        private int weight;
        //生日
        private Date birthDate;
        //金箍棒
        private GoldRingedStaff staff;
        /**
         * 构造函数
         */
        public Monkey(){
            this.birthDate = new Date();
            this.staff = new GoldRingedStaff();
        }
        /**
         * 克隆方法
         */
        public Object clone(){
            Monkey temp = null;
            try {
                temp = (Monkey) super.clone();
            } catch (CloneNotSupportedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            } finally {
                return temp;
            }
        }
        public int getHeight() {
            return height;
        }
        public void setHeight(int height) {
            this.height = height;
        }
        public int getWeight() {
            return weight;
        }
        public void setWeight(int weight) {
            this.weight = weight;
        }
        public Date getBirthDate() {
            return birthDate;
        }
        public void setBirthDate(Date birthDate) {
            this.birthDate = birthDate;
        }
        public GoldRingedStaff getStaff() {
            return staff;
        }
        public void setStaff(GoldRingedStaff staff) {
            this.staff = staff;
        }

    }

　　大圣还持有一个金箍棒的实例，金箍棒类GoldRingedStaff:

    public class GoldRingedStaff {
        private float height = 100.0f;
        private float diameter = 10.0f;
        /**
         * 增长行为，每次调用长度和半径增加一倍
         */
        public void grow(){
            this.diameter *= 2;
            this.height *= 2;
        }
        /**
         * 缩小行为，每次调用长度和半径减少一半
         */
        public void shrink(){
            this.diameter /= 2;
            this.height /= 2;
        }
    }

　　当运行TheGreatestSage类时，首先创建大圣本尊对象，而后浅度克隆大圣本尊对象。程序在运行时打印出的信息如下：
  
 ![下载 (1).png]({{site.baseurl}}/img/下载 (1).png)


　　可以看出，首先，复制的大圣本尊具有和原始的大圣本尊对象一样的birthDate，而本尊对象不相等，这表明他们二者是克隆关系；其次，复制的大圣本尊所持有的金箍棒和原始的大圣本尊所持有的金箍棒为同一个对象。这表明二者所持有的金箍棒根本是一根，而不是两根。

　　正如前面所述，继承自java.lang.Object类的clone()方法是浅克隆。换言之，齐天大圣的所有化身所持有的金箍棒引用全都是指向一个对象的，这与《西游记》中的描写并不一致。要纠正这一点，就需要考虑使用深克隆。

　　为做到深度克隆，所有需要复制的对象都需要实现java.io.Serializable接口。

　　孙大圣的源代码：


    public class TheGreatestSage {
        private Monkey monkey = new Monkey();

        public void change() throws IOException, ClassNotFoundException{
            Monkey copyMonkey = (Monkey)monkey.deepClone();
            System.out.println("大圣本尊的生日是：" + monkey.getBirthDate());
            System.out.println("克隆的大圣的生日是：" + monkey.getBirthDate());
            System.out.println("大圣本尊跟克隆的大圣是否为同一个对象 " + (monkey == copyMonkey));
            System.out.println("大圣本尊持有的金箍棒 跟 克隆的大圣持有的金箍棒是否为同一个对象？ " + (monkey.getStaff() == copyMonkey.getStaff()));
        }

        public static void main(String[]args) throws IOException, ClassNotFoundException{
            TheGreatestSage sage = new TheGreatestSage();
            sage.change();
        }
    }

　　在大圣本尊Monkey类里面，有两个克隆方法，一个是clone()，也即浅克隆；另一个是deepClone()，也即深克隆。在深克隆方法里，大圣本尊对象（一个拷贝）被序列化，然后又被反序列化。反序列化的对象就成了一个深克隆的结果。

    public class Monkey implements Cloneable,Serializable {
        //身高
        private int height;
        //体重
        private int weight;
        //生日
        private Date birthDate;
        //金箍棒
        private GoldRingedStaff staff;
        /**
         * 构造函数
         */
        public Monkey(){
            this.birthDate = new Date();
            staff = new GoldRingedStaff();
        }
        /**
         * 克隆方法
         */
        public Object clone(){
            Monkey temp = null;
            try {
                temp = (Monkey) super.clone();
            } catch (CloneNotSupportedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            } finally {
                return temp;
            }
        }
        public  Object deepClone() throws IOException, ClassNotFoundException{
            //将对象写到流里
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(bos);
            oos.writeObject(this);
            //从流里读回来
            ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
            ObjectInputStream ois = new ObjectInputStream(bis);
            return ois.readObject();
        }
        public int getHeight() {
            return height;
        }
        public void setHeight(int height) {
            this.height = height;
        }
        public int getWeight() {
            return weight;
        }
        public void setWeight(int weight) {
            this.weight = weight;
        }
        public Date getBirthDate() {
            return birthDate;
        }
        public void setBirthDate(Date birthDate) {
            this.birthDate = birthDate;
        }
        public GoldRingedStaff getStaff() {
            return staff;
        }
        public void setStaff(GoldRingedStaff staff) {
            this.staff = staff;
        }

    }

　　可以看到，大圣本尊持有一个金箍棒（GoldRingedStaff）的实例。在大圣复制件里面，此金箍棒实例是原大圣本尊对象所持有的金箍棒对象的一个拷贝。在大圣本尊对象被序列化和反序列化时，它所持有的金箍棒对象也同时被序列化和反序列化，这使得复制的大圣的金箍棒和原大圣本尊对象所持有的金箍棒对象是两个独立的对象。

    public class GoldRingedStaff implements Serializable{
        private float height = 100.0f;
        private float diameter = 10.0f;
        /**
         * 增长行为，每次调用长度和半径增加一倍
         */
        public void grow(){
            this.diameter *= 2;
            this.height *= 2;
        }
        /**
         * 缩小行为，每次调用长度和半径减少一半
         */
        public void shrink(){
            this.diameter /= 2;
            this.height /= 2;
        }
    }

　　运行结果：
 
 ![下载.png]({{site.baseurl}}/img/下载.png)


　　从运行的结果可以看出，大圣的金箍棒和他的身外之身的金箍棒是不同的对象。这是因为使用了深克隆，从而把大圣本尊所引用的对象也都复制了一遍，其中也包括金箍棒。
