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

因为以上优点，所以在需要重复地创建相似对象时可以考虑使用原型模式。比如需要在一个循环体内创建对象，假如对象创建过程比较复杂或者循环次数很多的话，使用原型模式不但可以简化创建过程，而且可以使系统的整体性能提高很多