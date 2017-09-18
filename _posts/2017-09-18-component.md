---
layout: post
published: true
title: 'Component组合模式 '
---
# Component组合模式 

将对象组合成树形结构以表示“部分-整体”的层次结构。
组合模式使得用户对单个对象和组合对象的使用具有唯一性。
组合模式是一种结构型模式。

## 角色：

　　1.Component 是组合中的对象声明接口，在适当的情况下，实现所有类共有接口的默认行为。声明一个接口用于访问和管理Component
子部件。

　　2.Leaf 在组合中表示叶子结点对象，叶子结点没有子结点。
  
　　3.Composite 定义有枝节点行为，用来存储子部件，在Component接口中实现与子部件有关操作，如增加(add)和删除(remove)等。

## 使用情况

引用大话设计模式的片段：“当发现需求中是体现部分与整体层次结构时，以及你希望用户可以忽略组合对象与单个对象的不同，统一地使用组合结构中的所有对象时，就应该考虑组合模式了。”
   
## DEMO

    public static void main(String[] args) {

            Component component=new Composite("根节点");
            Component child=new Composite("一级子节点child");
            Component child_1=new Leaf("一级子节点child之子节点一");
            Component child_2=new Leaf("一级子节点child之子节点二");
            child.add(child_1);
            child.add(child_2);
            Component child2=new Composite("一级子节点child2");
            component.add(child);
            component.add(child2);
            component.foreach();
        }

    }
     abstract class Component {  

         String name;

         public Component(String s){

             this.name=s;
         }
         public abstract void add(Component c);
         public abstract void remove(Component c);
         public abstract void foreach();
     }
    //组合类
     class Composite extends Component{
        private List<Component>child=new ArrayList<Component>
        ();

        public Composite(String s) {
            super(s);
            // TODO Auto-generated constructor stub
        }

        @Override
        public void add(Component c) {
            child.add(c);

        }

        @Override
        public void foreach() {
            // TODO Auto-generated method stub
            System.out.println("节点名:\t"+name);
            for (Component c : child) {
                c.foreach();
            }
        }

        @Override
        public void remove(Component c) {
            // TODO Auto-generated method stub
            child.remove(c);
        }

     }
     //不在有根节点
     class Leaf extends Component{

        public Leaf(String s) {
            super(s);

        }

        @Override
        public void add(Component c) {
            // TODO Auto-generated method stub

        }
        @Override
        public void foreach() {
            System.out.println("tself name-->"+this.name);
        }

        @Override
        public void remove(Component c) {
            // TODO Auto-generated method stub

        }

     }  
     
执行结果:

    节点名: 根节点
    节点名:	一级子节点child
    tself name-->一级子节点child之子节点一
    tself name-->一级子节点child之子节点二
    节点名:	一级子节点child2
     