# 设计模式

## 单例模式
单例模式的主要作用是保证在Java程序中，某个类只有一个实例存在。
* 饿汉模式（线程安全）
```java
public class Singleton {
    private static final Singleton instance = new Singleton();
    private Singleton(){}
    public static Singleton newInstance(){
        return instance;
    }
}
```
* 懒汉模式
```java
public class Singleton {
    private static Singleton instance = null;
    private Singleton(){}
    public static Singleton newInstance(){
        if(null == instance){
            instance = new Singleton();
        }
        return instance;
    }
}
```
* 双重校验锁-懒汉模式（线程安全）
```java
public class Singleton {
    private static volatile Singleton instance = null;
    private Singleton(){}
    public static Singleton getInstance() {
        if (instance == null) {   // Single Checked
            synchronized (Singleton.class) {
                if (instance == null) { // Double checked
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```
注意：instance使用volatile关键字修饰，避免指令重排，对instance的写操作对所有线程立即生效的。

* 静态内部类（线程安全）
```java
public class Singleton {
    private static class SingletonHolder {
        public static final Singleton instance = new Singleton();
    }
    private Singleton(){}
    public static Singleton newInstance(){
        return SingletonHolder.instance;
    }
}
```
利用了类加载机制来实现懒汉模式，同时保证了线程安全



## 原型模式
定义：用原型实例指定创建对象的种类，并通过拷贝这些原型创建新的对象   
类型：创建类模式

使用原型模式创建对象比直接new一个对象在性能上要好的多，因为Object类的clone方法是一个本地方法，它直接操作内存中的二进制流，特别是复制大对象时，性能的差别非常明显。

使用原型模式的另一个好处是简化对象的创建，使得创建对象就像我们在编辑文档时的复制粘贴一样简单。

因为以上优点，所以在需要重复地创建相似对象时可以考虑使用原型模式。比如需要在一个循环体内创建对象，假如对象创建过程比较复杂或者循环次数很多的话，使用原型模式不但可以简化创建过程，而且可以使系统的整体性能提高很多。

## 工厂模式
<img src='https://g.gravizo.com/svg?
    /**@opt all*/
    interface IProduct {
        public void productMethod%28%29;
    }
    class Product implements IProduct {
        public void productMethod%28%29 {
            System.out.println%28"产品"%29;
        }
    }
    /**@opt all*/
    interface IFactory {
        public IProduct createProduct%28%29;
    }
    class Factory implements IFactory {
        public IProduct createProduct%28%29 {
            return new Product%28%29;
        }
    }
'>

降低耦合度；对调用者屏蔽具体的产品类；可以使代码结构清晰，有效地封装变化。

## 责任链模式  
客户端发出一个请求，链上的对象都有机会来处理这一请求，而客户端不需要知道谁是具体的处理对象。这样就实现了请求者和接受者之间的解耦，并且在客户端可以实现动态的组合职责链。使编程更有灵活性。

角色：  
`Handler`：定义职责接口，通常在内部定义处理请求的方法，可以在这里实现后继链。  
`ConcreteHandler`：实际的职责类，在这里个类里面，实现在它职责范围内的请求处理，如果不处理，就继续转发请求给后继者。  
`Client`：客户端，组装职责链，向链上的具体对象提交请求  

应用：  
* View事件的分发处理  
ViewGroup事件投递的递归调用就类似于一条责任链，一旦其寻找到责任者，那么将由责任者持有并消费掉该次事件，具体体现在View的onTouchEvent方法中返回值的设置，如果返回false，那么意味着当前的View不会是该次的责任人，将不会对其持有；如果返回true，此时View会持有该事件并不再向外传递。
* Broadcast广播机制

对于责任链中的一个处理者对象，有两个行为。一是处理请求，二是将请求传递到下一节点，不允许某个处理者对象在处理了请求后又将请求传送给上一个节点的情况


## 模板方法模式
在模板模式（Template Pattern）中，一个抽象类公开定义了执行它的方法的方式/模板。它的子类可以按需要重写方法实现，但调用将以抽象类中定义的方式进行。这种类型的设计模式属于行为型模式。

<img src='https://g.gravizo.com/svg?
    /**@opt all*/
    interface Game {
        public void initialize%28%29;
        public void startPlay%28%29;
        public void endPlay%28%29;
    }
    class KingGloyGame implements Game {
        public void productMethod%28%29 {
            System.out.println%28"产品"%29;
        }
    }
    class LoLGame implements Game {
        public void productMethod%28%29 {
            System.out.println%28"产品"%29;
        }
    }
'>