# AspectJ

## 是什么？
AspectJ是Java中流行的AOP（Aspect-oriented Programming）编程扩展框架，是Eclipse托管给Apache基金会的一个开源项目。
AspectJ在Java开发平台，一般使用AspectJ编译器(ajc)织入AOP代码。  
而在Android开发平台，无法直接使用ajc，而需要使用封装ajc的[AspectJX编译插件](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)，它的工作原理是：通过Gradle Transform，调用ajc，在class文件生成后至dex文件生成前，遍历并匹配所有符合AspectJ文件中声明的切点，然后将事先声明好的代码在切点前后织入。  
由于整个过程发生在编译期，是一种`静态织入`方式，所以会增加一定的编译时长，但几乎不会影响程序的运行时效率。

## 做什么？
AspectJ主要用来实现AOP，通常来说，AOP都是为一些相对基础且固定的需求服务，实际常见的场景大致包括：
```
统计埋点
日志打印/打点
数据校验
行为拦截
性能监控
动态权限控制
```
如果你在项目中也有这样的需求，可以考虑通过AspectJ来实现。

## 怎么做？
### 关于AOP的一些概念
* 切面（Aspect）：一个关注点的模块化，这个关注点实现可能另外横切多个对象。其实就是共有功能的实现。如日志切面、权限切面、事务切面等。  
* 通知（Advice）：是切面的具体实现。以目标方法为参照点，根据放置的地方不同，可分为前置通知（Before）、后置通知（AfterReturning）、异常通知（AfterThrowing）、最终通知（After）与环绕通知（Around）5种。在实际应用中通常是切面类中的一个方法，具体属于哪类通知由配置指定的。  
* 切入点（Pointcut）：用于定义通知应该切入到哪些连接点上。不同的通知通常需要切入到不同的连接点上，这种精准的匹配是由切入点的正则表达式来定义的。  
* 连接点（Joinpoint）：就是程序在运行过程中能够插入切面的地点。例如，方法调用、异常抛出或字段修改等。

### AspectJ基本语法
```
@Aspect 用它声明一个类，表示一个需要执行的切面配置。
@Pointcut 声明一个切点。
@Before/@After/@Around/...（统称为Advice类型） 声明在切点前、后、中执行切面代码。
```

### Join Point
Join Points是AspectJ中的一个关键概念。Join Points可以看作是程序运行时的一个执行点。一个程序中，哪些执行点可以是Join Points 呢？
* 一个函数的调用可以是一个Join Point。
    比如Log.e()这个函数。e的执行可以是一个Join Point，而调用e的函数也可以认为是一个Join Point。
* 设置一个变量，或者读取一个变量，也可以是一个Join Point。
    比如某个类中有一个debug的boolean变量。设置它的地方或者读取它的地方都可以看做是Join Point。
* for循环也可以看做是Join Point。
* …
理论上说，一个程序中很多地方都可以被看做是Join Points，但是AspectJ中，只有下面所示的几种执行点被认为是Join Points：
Join Point|Pointcut|Pointcut表达式
---|----|----
method call|函数被调用|call(MethodSignature)
method execution|函数执行|execution(MethodSignature)
constructor call|构造函数调用|call(ConstructorSignature)
constructor execution|构造函数执行|execution(ConstructorSignature)
field get|获取某个变量|get(FieldSignature)
field set|设置某个变量|set(FieldSignature)
pre-initialization|与构造函数有关，很少用到|preinitialization(ConstructorSignature)
initialization|与构造函数有关，很少用到|initialization(ConstructorSignature)
static initialization|static块初始化|staticinitialization(TypeSignature)
handler|异常处理|handler(TypeSignature)注：只能与@Before()配合使用
advice execution|advice执行|adviceexecution()

以上列出了AspectJ所认可的JoinPoints的类型。实际上就是你想把新的代码插在程序的哪个地方，是插在构造方法中，还是插在某个方法调用前，或者是插在某个方法中，这个地方就是Join Points。

**call 和 execution的区别是：call 是方法被调用位置；而execution是方法体内的执行位置。**
```java
//call(* *..*.Animal.fly(..))
Animal a = new Animal();
//...我是织入代码
a.fly();

//execution(* *..*.Animal.fly(..))
public class Animal {
    public void fly() {
        //...我是织入代码
        Log.e(TAG, "animal fly");
    }
}
```

### Pointcuts
一个程序会有很多的JPoints，即使是同一个函数，还分为call类型和execution类型的JPoint。显然，不是所有的JPoint，也不是所有类型的JPoint都是我们关注的。那么怎么从一堆一堆的JPoints中选择自己想要的JPoints呢？这就是Pointcuts的功能：提供一种方法使得开发者能够选择自己感兴趣的Join Points。
```java
@Pointcut("execution(@com.kun.aspectjtest.aspect.DebugTrace * *..*.*(..))")
public void DebugTraceMethod() {}
```
@Pointcut，表示这里定义的是一个Pointcut，在AspectJ中，你总是得定义一个Pointcut。
本例中，execution(xxx)是一种选择条件。execution表示我们选择的Joinpoint类型为execution类型。

Signature|语法(间隔一个空格)
---|----
MethodSignature|[@Annotation] [public,protected,private] [static] [final] 返回值类型 [类名.]方法名(参数类型列表) [throws 异常类型]
ConstructorSignature|[@Annotation] [public,protected,private] [final] [类名.]new(参数类型列表) [throws 异常类型]
FieldSignature|[@Annotation] [public,protected,private] [static] [final] 属性类型 [类名.]属性名
TypeSignature|其他 Pattern 涉及到的类型规则也是一样，可以使用 !、\*、.、..、+、!表示取反，* 匹配除 . 外的所有字符串，* 单独使用表示匹配任意类型，.. 匹配任意字符串，.. 单独使用时表示匹配任意长度任意类型，+ 匹配其自身及子类，还有一个 ... 表示不定个数

示例：
1. (int,char)表示参数为int和char
2. (String,..)，表示第一个参数是String，后面随意
3. (String ...) 表示参数个数不定，且都是String类型

### Advice
我们知道如何通过Pointcuts来选择合适的JPoint。那么下一步工作就很明确了，选择这些JPoint后，我们肯定是需要干一些事情的。  
Advice其实是最好理解的，也就是我们具体插入的代码，以及如何插入这些代码。以目标方法为参照点，根据放置的地方不同，可分为前置通知（Before）、后置通知（AfterReturning）、异常通知（AfterThrowing）、最终通知（After）与环绕通知（Around）5种。在AspectJ中有五种类型的注解：Before、After、AfterReturning、AfterThrowing、Around，我们将它们统称为Advice注解。
Advice|说明
---|---
@Before(Pointcut)|执行JPoint前
@After(Pointcut)|执行JPoint后
@AfterReturning|@AfterReturning(pointcut = “xxx", returning = “retValue")
@AfterThrowing|@AfterThrowing(pointcut = “xxx", throwing = “throwable")
@Around(Pointcut)|替代原来的代码，如果要执行原代码，需要使用ProceedingJoinPoint#proceed()，注：不支持和@Before()和@After()一起使用。

Before和After其实还是很好理解的，也就是在Pointcuts之前和之后，插入代码。
```java
//在activty的任意onxxx方法开始执行前插入以下代码
@Before("execution(* android.app.Activity.on**(..))")
public void onActivityMethodBefore(JoinPoint joinPoint) throws Throwable {
    
}
```
* @Before：Advice，也就是具体的插入点。
* execution：处理Join Point的类型，例如call、execution。
* (\* android.app.Activity.on\*\*(..))：这个是最重要的表达式，第一个\*表示返回值，\*表示返回值为任意类型，后面这个就是典型的包名路径，其中可以包含 * 来进行通配，几个 * 没区别。同时，这里可以通过 &&、||、!来进行条件组合。()代表这个方法的参数，你可以指定类型，例如android.os.Bundle，或者(..)这样来代表任意类型、任意个数的参数。
* public void onActivityMethodBefore：实际切入的代码。

Around从字面含义上来讲，也就是在方法前后各插入代码，实际上它是替换原有逻辑，当然原有逻辑通过proceedingJoinPoint.proceed()可得到执行。
```java
@Around("execution(* com.test.MainActivity.testAOP())")
public void onActivityMethodAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
    String key = proceedingJoinPoint.getSignature().toString();
    Log.d(TAG, "onActivityMethodAroundFirst: " + key);
    proceedingJoinPoint.proceed();//执行原有逻辑
    Log.d(TAG, "onActivityMethodAroundSecond: " + key);
}
```
Advice注解修饰的方法有一些约束：  
* 方法必须为public。
* Before、After、AfterReturning、AfterThrowing 四种类型方法返回值必须为void。
* Around的目标是替代原切入点，它一般会有返回值，这就要求声明的返回值类型必须与切入点方法的返回值保持一致；不能和其他 Advice 一起使用，如果在对一个 Pointcut 声明 Around 之后还声明 Before 或者 After 则会失效。

除了上面与 Join Point 对应的选择外，Pointcuts 还有其他选择方法。
Pointcuts 表达式|说明
---|---
within(TypePattern)|符合 TypePattern 的代码中的 Join Point
withincode(MethodPattern)|在某些方法中的 Join Point
withincode(ConstructorPattern)|在某些构造函数中的 Join Point
cflow(Pointcut)|Pointcut 选择出的切入点 P 的控制流中的所有 Join Point，包括 P 本身
cflowbelow(Pointcut)|Pointcut 选择出的切入点 P 的控制流中的所有 Join Point，不包括 P 本身
this(Type or Id)|Join Point 所属的 this 对象是否 instanceOf Type 或者 Id 的类型
target(Type or Id)|Join Point 所在的对象（例如 call 或 execution 操作符应用的对象）是否 instanceOf Type 或者 Id 的类型
args(Type or Id, ...)|方法或构造函数参数的类型
if(BooleanExpression)|满足表达式的 Join Point，表达式只能使用静态属性、Pointcuts 或 Advice 暴露的参数、thisJoinPoint 对象

### 自定义PointCut
使用Pointcut注解来定义一个Pointcut。
```java
//定义使用注解DebugTrace的任意方法为一个Pointcut：DebugTraceMethod
@Pointcut("execution(@com.kun.aspectjtest.aspect.DebugTrace * *..*.*(..))")
public void DebugTraceMethod() {}

//在以上Pointcut执行前插入以下方法调用
@Before("DebugTraceMethod()")
public void beforeDebugTraceMethod(JoinPoint joinPoint) throws Throwable {
    String key = joinPoint.getSignature().toString();
    Log.e(TAG, "beforeDebugTraceMethod: " + key);
}
```

### 条件运算
 Pointcut表达式中还可以使用一些条件判断符，比如 ！、&&、||。
```java
# Hugo.java
@Pointcut("within(@hugo.weaving.DebugLog *)")
public void withinAnnotatedClass() {}

@Pointcut("execution(!synthetic * *(..)) && withinAnnotatedClass()")
public void methodInsideAnnotatedType() {}
```
第一个切点指定范围为包含DebugLog注解的任意类和方法，第二个切点为在第一个切点范围内，且执行非内部类的任意方法。结合起来表述就是任意声明了DebugLog注解的方法。

```java
//执行 Fragment 及其子类的 setUserVisibleHint(boolean) 方法时。
execution(void setUserVisibleHint(..)) && target(android.support.v4.app.Fragment) && args(boolean) 
//执行 Foo.foo() 方法中再递归执行 Foo.foo() 时。
execution(void Foo.foo(..)) && cflowbelow(execution(void Foo.foo(..))) 
```

### JoinPoint方法
1. 获取方法名
```java
((MethodSignature) joinPoint.getSignature()).getName()
```
2. 获取this
this指代的是被织入代码所属类的实例对象。
```java
joinPoint.getThis()
```
3. 获取targe
targe是切入点方法的所有者。
```java
joinPoint.getTarget()
```
4. 获取方法参数
```java
joinPoint.getArgs()
```

### 传递参数
前面介绍的advice都是没有参数信息的，而JPoint肯定是或多或少有参数的。而且advice既然是对JPoint的截获或者hook也好，肯定需要利用传入给JPoint的参数干点什么事情。比方所around advice，我可以对传入的参数进行检查，如果参数不合法，我就直接返回，根本就不需要调用proceed做处理。

往advice传参数比较简单，就是利用前面提到的this(),target(),args()等方法。另外整个pointcuts和advice编写的语法也有一些区别。具体方法如下：
先在pointcuts定义时候指定参数类型和名字
```java
Before("execution(* Test.TestDerived.testMethod(..)) && args(x) && target(derived)")
public void testAll(int x, TestDerived derived) {
    System.out.println("arg1=" + derived);  
    System.out.println("arg2=" + x);  
 }
 ```
注意上述pointcuts的写法，首先在testAll中定义参数类型和参数名。这一点和定义一个函数完全一样，接着看target和args。此处的target和args括号中用得是参数名。而参数名则是在前面pointcuts中定义好的。这属于target和args的另外一种用法。
**注意，增加参数并不会影响pointcuts对JPoint的匹配**

而advice的定义现在也和函数定义一样，把参数类型和参数名传进来。
**注意，advice必须和使用的pointcuts在参数类型和名字上保持一致。**

总结，参数传递其实并不复杂，关键是得记住语法：
1. pointcuts修改：像定义函数一样定义pointcuts，然后在this,target或args中绑定参数名（注意，不再是参数类型，而是参数名）
2. advice修改：也像定义函数一样定义advice，然后在函数参数列表中绑定参数名（注意是参数名）
3. 在advice的代码中使用参数名

### 问题
1. 重复织入、不织入
假如我们想对Activity生命周期织入埋点统计，我们可能写出这样的切点代码。
```java
@Pointcut("execution(* android.app.Activity+.on*(..))")
public void callMethod() {}
```
由于Activity.class不参与打包，参与打包是那些支持库比如support-v7中的AppCompatActivity，还有项目里定义的Activity，这就导致了：  
* 如果我们业务Activity中如果没有复写生命周期方法将不会织入。
* 如果我们的Activity继承树上如果都复写了生命周期方法，那么继承树上的所有Activity都会织入统计代码，这会导致重复统计。

2. 出问题难排查
这是AOP技术的实现方式决定的，修改字节码过程，对上层应用无感知，容易将问题隐藏，排查难度大。因此如果项目中使用了AOP技术应当完善文档，并知会协同开发人员。

3. 编译时间变长

### 实践
[使用AspectJ实现android权限申请](https://github.com/qylk/AspectJTest_Permission)

## 附录
[AspectJ语法](http://www.eclipse.org/aspectj/doc/released/progguide/semantics-pointcuts.html)
[AOP 之 AspectJ 全面剖析 in Android](https://www.jianshu.com/p/f90e04bcb326)