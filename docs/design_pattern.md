# 设计模式
## 六大原则
### 单一职责原则
单一原则很简单，就是将一组相关性很高的函数、数据封装到一个类中。换句话说，一个类应该有职责单一。

### 开闭原则
开闭原则理解起来也不复杂，就是一个类应该对于扩展是开放的，但是对于修改是封闭的。在一开始编写代码时，就应该注意尽量通过扩展的方式实现新的功能，而不是通过修改已有的代码实现，否则容易破坏原有的系统，也可能带来新的问题，如果发现没办法通过扩展来实现，应该考虑是否是代码结构上的问题，通过重构等方式进行解决。

### 里氏替换原则
所有引用基类的地方必须能透明地使用其子类对象。本质上就是说要好好利用继承和多态，从而以父类的形式来声明变量（或形参），为变量（或形参）赋值任何继承于这个父类的子类。

### 依赖倒置原则
依赖倒置主要是实现解耦，使得高层次的模块不依赖于低层次模块的具体实现细节。怎么去理解它呢，我们需要知道几个关键点：

* 高层模块不应该依赖底层模块（具体实现），二者都应该依赖其抽象（抽象类或接口）
* 抽象不应该依赖细节
* 细节应该依赖于抽象
在我们用的Java语言中，抽象就是指接口或者抽象类，二者都是不能直接被实例化；细节就是实现类，实现接口或者继承抽象类而产生的类，就是细节。使用Java语言描述就是：各个模块之间相互传递的参数声明为抽象类型，而不是声明为具体的实现类；

### 接口隔离原则
类之间的依赖关系应该建立在最小的接口上。其原则是将非常庞大的、臃肿的接口拆分成更小的更具体的接口。

### 迪米特原则
一个对象应该对其他的对象有最少的了解.
假设类A实现了某个功能，类B需要调用类A的去执行这个功能，那么类A应该只暴露一个函数给类B，这个函数表示是实现这个功能的函数，而不是让类A把实现这个功能的所有细分的函数暴露给B。

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

Android上的应用：
```java
WindowManager wm = (WindowManager)getSystemService(getApplication().WINDOW_SERVICE);
```

## 建造者模式
将一个复杂对象的构造与它的表示分离，使得同样的构造过程可以创建不同的表示。
主要是在创建某个对象时，需要设定很多的参数（通过setter方法），但是这些参数必须按照某个顺序设定，或者是设置步骤不同会得到不同结果。

Android上的应用：
```java
AlertDialog.Builer builder=new AlertDialog.Builder(context);
builder.setIcon(R.drawable.icon)
    .setTitle("title")
    .setMessage("message")
    .setPositiveButton("Button1",
        new DialogInterface.OnclickListener(){
            public void onClick(DialogInterface dialog,int whichButton){
                setTitle("click");
            }   
        })
    .create()
    .show();
```

## 原型模式
定义：用原型实例指定创建对象的种类，并通过拷贝这些原型创建新的对象   
类型：创建类模式

使用原型模式创建对象比直接new一个对象在性能上要好的多，因为Object类的clone方法是一个本地方法，它直接操作内存中的二进制流，特别是复制大对象时，性能的差别非常明显。

使用原型模式的另一个好处是简化对象的创建，使得创建对象就像我们在编辑文档时的复制粘贴一样简单。

因为以上优点，所以在需要重复地创建相似对象时可以考虑使用原型模式。比如需要在一个循环体内创建对象，假如对象创建过程比较复杂或者循环次数很多的话，使用原型模式不但可以简化创建过程，而且可以使系统的整体性能提高很多。

Android上的应用：
```java
Intent intent = new Intent(Intent.ACTION_SENDTO, uri);
//克隆副本
Intent copyIntent = (Intetn) shareIntent.clone();
```

## 工厂模式
### 简单工厂模式
建立一个工厂（一个函数或一个类方法）来制造新的对象。

Android上的应用：
```java
public Object getSystemService(String name) {
    if (getBaseContext() == null) {
        throw new IllegalStateException("System services not available to Activities before onCreate()");
    }
    //........
    if (WINDOW_SERVICE.equals(name)) {
         return mWindowManager;
    } else if (SEARCH_SERVICE.equals(name)) {
        ensureSearchManager();
        return mSearchManager;
    }
    //.......
    return super.getSystemService(name);
}
```

### 工厂方法模式
定义一个创建产品对象的工厂接口，让其子类决定实例化哪一个类，将实际创建工作推迟到子类当中。

降低耦合度；对调用者屏蔽具体的产品类；可以使代码结构清晰，有效地封装变化。

实例：
```java
public abstract class Product {
    public abstract void make();
}

public class ConcreteProduct extends Prodect {
    public void make(){
        System.out.println("我是具体产品！");
    }
}

public  abstract class Factory {
    public abstract Product createProduct();
}

public class ConcreteFactory extends Factory{

    public Product createProduct(){
        return new ConcreteProduct();
    }
}

Factory factory = new ConcreateFactory();
Product product = factory.createProduct();
product.method();
```

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

## 责任链模式  
客户端发出一个请求，链上的对象都有机会来处理这一请求，而客户端不需要知道谁是具体的处理对象。这样就实现了请求者和接受者之间的解耦，并且在客户端可以实现动态的组合职责链。使编程更有灵活性。

角色：  
`Handler`：定义职责接口，通常在内部定义处理请求的方法，可以在这里实现后继链。  
`ConcreteHandler`：实际的职责类，在这里个类里面，实现在它职责范围内的请求处理，如果不处理，就继续转发请求给后继者。  
`Client`：客户端，组装职责链，向链上的具体对象提交请求  

Android上的应用：  
* View事件的分发处理  
ViewGroup事件投递的递归调用就类似于一条责任链，一旦其寻找到责任者，那么将由责任者持有并消费掉该次事件，具体体现在View的onTouchEvent方法中返回值的设置，如果返回false，那么意味着当前的View不会是该次的责任人，将不会对其持有；如果返回true，此时View会持有该事件并不再向外传递。
* Broadcast广播机制

对于责任链中的一个处理者对象，有两个行为。一是处理请求，二是将请求传递到下一节点，不允许某个处理者对象在处理了请求后又将请求传送给上一个节点的情况


## 策略模式
策略模式让算法独立于使用它的客户而独立变化。
```java
public abstract class BaseAdvertManager {
    protected abstract void doLoadAdvert();
}

public class FacebookAdvertManager extends BaseAdvertManager {
 @Override
    protected void doLoadAdvert() {
        Log.v(TAG, "加载Facebook广告");
    }
}

public class AdmobAdvertManager extends BaseAdvertManager {
 @Override
    protected void doLoadAdvert() {
        Log.v(TAG, "加载Admob广告");
    }
}
```

Android上的应用：  
Android在属性动画中使用时间插值器的时候就用到了策略模式。在使用动画时，你可以选择线性插值器LinearInterpolator、加速减速插值器AccelerateDecelerateInterpolator、减速插值器DecelerateInterpolator以及自定义的插值器。这些插值器都是实现根据时间流逝的百分比来计算出当前属性值改变的百分比。通过根据需要选择不同的插值器，实现不同的动画效果。

## 状态模式
状态模式中，行为是由状态来决定的，不同状态下有不同行为。状态模式和策略模式的结构几乎是一模一样的，主要是他们表达的目的和本质是不同。
实例：
```java
public interface TvState{
    public void nextChannerl();
    public void prevChannerl();
    public void turnUp();
    public void turnDown();
}

public class PowerOffState implements TvState{
    public void nextChannel(){}
    public void prevChannel(){}
    public void turnUp(){}
    public void turnDown(){}

}

public class PowerOnState implements TvState{
    public void nextChannel(){
        System.out.println("下一频道");
    }
    public void prevChannel(){
        System.out.println("上一频道");
    }
    public void turnUp(){
        System.out.println("调高音量");
    }
    public void turnDown(){
        System.out.println("调低音量");
    }

}

public interface PowerController{
    public void powerOn();
    public void powerOff();
}

public class TvController implements PowerController{
    TvState mTvState;
    public void setTvState(TvStete tvState){
        mTvState=tvState;
    }
    public void powerOn(){
        setTvState(new PowerOnState());
        System.out.println("开机啦");
    }
    public void powerOff(){
        setTvState(new PowerOffState());
        System.out.println("关机啦");
    }
    public void nextChannel(){
        mTvState.nextChannel();
    }
    public void prevChannel(){
        mTvState.prevChannel();
    }
    public void turnUp(){
        mTvState.turnUp();
    }
    public void turnDown(){
        mTvState.turnDown();
    }
}


public class Client{
    public static void main(String[] args){
        TvController tvController=new TvController();
        tvController.powerOn();
        tvController.nextChannel();
        tvController.turnUp();

        tvController.powerOff();
        //调高音量，此时不会生效
        tvController.turnUp();
    }
}
```

**与策略模式的区别**  
状态模式的行为是平行的、不可替换的，策略模式是属于对象的行为模式，其行为是彼此独立可相互替换的。


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

Android上的应用：  
启动一个Activity过程非常复杂，有很多地方需要开发者定制，也就是说，整体算法框架是相同的，但是将一些步骤延迟到子类中，比如Activity的onCreate、onStart等等。这样子类不用改变整体启动Activity过程即可重定义某些具体的操作了。


## 解释器模式
给定一个语言，定义它的语法，并定义一个解释器，这个解释器用于解析语言。

Android上的应用：  
这个用到的地方也不少，其一就是Android的四大组件需要在AndroidManifest.xml中定义，其实AndroidManifest.xml就定义了，等标签（语句）的属性以及其子标签，规定了具体的使用（语法），通过PackageManagerService（解释器）进行解析。

## 命令模式
命令模式将每个请求封装成一个对象，从而让用户使用不同的请求把客户端参数化；将请求进行排队或者记录请求日志，以及支持可撤销操作。

举个例子来理解：当我们点击“关机”命令，系统会执行一系列操作，比如暂停事件处理、保存系统配置、结束程序进程、调用内核命令关闭计算机等等，这些命令封装从不同的对象，然后放入到队列中一个个去执行，还可以提供撤销操作。

## 观察者模式
有时被称作发布/订阅模式，其定义了一种一对多的依赖关系，让多个观察者对象同时监听某一个主题对象。这个主题对象在状态发生变化时，会通知所有观察者对象，使它们能够自动更新自己。

Android上的应用：  
ListView的适配器有个notifyDataSetChange()函数，就是通知ListView的每个Item，数据源发生了变化，各个子Item需要重新刷新一下。

EventBus

## 备忘录模式
在不破坏封闭的前提下，捕获一个对象的内部状态，并在对象之外保存这个状态，这样，以后就可将对象恢复到原先保存的状态中。

Android上的应用：  
Activity的onSaveInstanceState和onRestoreInstanceState就是用到了备忘录模式，分别用于保存和恢复。

## 迭代器模式
Android上的应用：  
Android源码中，最典型的就是Cursor用到了迭代器模式，当我们使用SQLiteDatabase的query方法时，返回的就是Cursor对象，之后再通过Cursor去遍历数据

## 访问者模式
Android上的应用： 
Android中运用访问者模式，其实主要是在编译期注解中，编译期注解核心原理依赖APT(Annotation Processing Tools)，著名的开源库比如ButterKnife、Dagger、Retrofit都是基于APT。

## 中介者模式
中介者模式包装了一系列对象相互作用的方式，使得这些对象不必相互明显调用，从而使他们可以轻松耦合。当某些对象之间的作用发生改变时，不会立即影响其他的一些对象之间的作用保证这些作用可以彼此独立的变化，中介者模式将多对多的相互作用转为一对多的相互作用。
其实，中介者对象是将系统从网状结构转为以调停者为中心的星型结构。

举个简单的例子，一台电脑包括：CPU、内存、显卡、IO设备。其实，要启动一台计算机，有了CPU和内存就够了。当然，如果你需要连接显示器显示画面，那就得加显卡，如果你需要存储数据，那就要IO设备，但是这并不是最重要的，它们只是分割开来的普通零件而已，我们需要一样东西把这些零件整合起来，变成一个完整体，这个东西就是主板。主板就是起到中介者的作用，任何两个模块之间的通信都会经过主板协调。

Android上的应用： 
在Binder机制中，就用到了中介者模式。我们知道系统启动时，各种系统服务会向ServiceManager提交注册，即ServiceManager持有各种系统服务的引用 ，当我们需要获取系统的Service时，比如ActivityManager、WindowManager等（它们都是Binder），首先是向ServiceManager查询指定标示符对应的Binder，再由ServiceManager返回Binder的引用。并且客户端和服务端之间的通信是通过Binder驱动来实现，这里的ServiceManager和Binder驱动就是中介者。

## 外观模式/门面模式
要求一个子系统的外部与其内部的通信必须通过一个统一的对象进行。

Android上的应用：  
Retrofit

## 代理模式
