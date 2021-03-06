# 概念：
APT(Annotation Processing Tool)即注解处理器，是一种处理注解的工具，确切的说它是javac的一个工具，它用来在编译时扫描和处理注解。

注解处理器以Java代码(或者编译过的字节码)作为输入，生成.java文件作为输出。 简单来说就是在编译期，可以通过注解生成.java文件。 

Android-apt是由一位开发者自己开发的apt框架，随着Android Gradle 插件 2.2 版本的发布，Android Gradle 插件提供了名为 annotationProcessor 的功能来完全代替 android-apt ，自此android-apt 作者在官网发表声明证实了后续将不会继续维护 android-apt ，并推荐大家使用 Android 官方插件 annotationProcessor。 AnnotationProcessor就是APT工具中的一种，是

google开发的内置框架，不需要引入。

# 自定义注解器
定义一个注解处理器，需要继承自AbstractProcessor。
```java
package com.example;

public class MyProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment env){ }

    @Override
    public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }

    @Override
    public Set<String> getSupportedAnnotationTypes() { }

    @Override
    public SourceVersion getSupportedSourceVersion() { }

}
```

* init(ProcessingEnvironment env): 每一个注解处理器类都必须有一个空的构造函数。然而这里有一个特殊的init()方法，它会被注解处理工具调用，并输入ProcessingEnviroment参数。ProcessingEnviroment提供很多有用的工具类如Elements, Types和Filer等。

* process(Set< ? extends TypeElement> annotations, RoundEnvironment env): 这相当于每个处理器的主函数main()。你在这里写你的扫描、评估和处理注解的代码，以及生成Java文件。输入参数RoundEnviroment，可以让你查询出包含特定注解的被注解元素。

* getSupportedAnnotationTypes(): 这里你必须指定，这个注解处理器是注册给哪个注解的。注意，它的返回值是一个字符串的集合，包含本处理器想要处理的注解类型的合法全称。

* getSupportedSourceVersion(): 用来指定你使用的Java版本。通常这里返回SourceVersion.latestSupported()，然而，如果你有足够的理由只支持Java 6的话，你也可以返回SourceVersion.RELEASE_6。  

在Java 7中，你也可以使用注解来代替getSupportedAnnotationTypes()和getSupportedSourceVersion()
```java
@SupportedSourceVersion(SourceVersion.latestSupported())
@SupportedAnnotationTypes({
   // 合法注解全名的集合
 })
public class MyProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment env){ }

    @Override
    public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }
}
```
另外需要知道的是：注解处理器是运行它自己的虚拟机JVM中，javac启动一个完整Java虚拟机来运行注解处理器。这对你意味着什么？你可以使用任何你在其他java应用中使用的的东西。  


# 注册注解处理器
如何将我们自定义的处理器MyProcessor注册到javac中呢？

首先我们需要将我们的注解处理器打包到一个jar文件中，其次在这个jar中，需要打包一个特定的文件 javax.annotation.processing.Processor 到META-INF/services路径下。

以下是这个jar的大致结构示意图：

    MyProcessor.jar 
        com 
            example 
                MyProcessor.jar 
    META-INF 
        services 
            javax.annotation.processing.Processor 


打包进MyProcessor.jar中的 javax.annotation.processing.Processor 的内容是注解处理器的合法的全名列表，每一个元素换行分割：

    com.example.MyProcessor  
    com.foo.OtherProcessor  
    net.blabla.SpecialProcessor 

把MyProcessor.jar放到你的build path中，javac会自动检查和读取javax.annotation.processing.Processor中的内容，并且注册MyProcessor等作为注解处理器。

Google提供了一个插件来帮助我们更方便的注册注解处理器，@AutoService注解可以方便的生成 META-INF/services/javax.annotation.processing.Processor中注解的配置信息。

你只需要导入对应的依赖包，在自定义的Processor类上方添加@AutoService(Processor.class)即可，如下:

* 导入依赖包
```groovy
compile 'com.google.auto.service:auto-service:1.0-rc2'
```
* 添加声明
```groovy
@AutoService(Processor.class)
public class MyProcessor extends AbstractProcessor { 
  ... 
}
```

# 使用注解处理器
添加处理声明：
```groovy
annotationProcessor project(':xxx-compiler')
```

# 实践例子
1、假设我们定义了一个注解，叫@Factory，用来生产一些产品。@Factory注解的参数是type和id，分别表示产品的具体类型和产品id。
```java
/**
元注解，是放在被定义的一个注解类的前面 ，是对注解一种限制。
@Target :  用来说明该注解可以被声明在哪些元素之前：
    ElementType.TYPE：说明该注解只能被声明在一个类前。
    ElementType.FIELD：说明该注解只能被声明在一个类的字段前。
    ElementType.METHOD：说明该注解只能被声明在一个类的方法前。
    ElementType.PARAMETER：说明该注解只能被声明在一个方法参数前。
    ElementType.CONSTRUCTOR：说明该注解只能声明在一个类的构造方法前。
    ElementType.LOCAL_VARIABLE：说明该注解只能声明在一个局部变量前。
    ElementType.ANNOTATION_TYPE：说明该注解只能声明在一个注解类型前。
    ElementType.PACKAGE：说明该注解只能声明在一个包名前。

@Retention ：用来说明该注解类的生命周期。它有以下三个参数：
    RetentionPolicy.SOURCE： 注解只保留在源文件中
    RetentionPolicy.CLASS： 注解保留在class文件中，在加载到JVM虚拟机时丢弃
    RetentionPolicy.RUNTIME：注解保留在程序运行期间，此时可以通过反射获得定义在某个类上的所有注解。
*/
@Target(ElementType.TYPE)       //该注解只能用在类上
@Retention(RetentionPolicy.CLASS)//该注解保留在class文件上
public @interface Factory {

  /**
   * 工厂的名字
   */
  Class type();

  /**
   * 用来表示生成哪个对象的唯一id
   */
  String id();
}
```

2、现在我们使用@Factory注解一些产品，假设这里我们有两个Meal类型的产品：MargheritaPizza（一种披萨）和Tiramisu（提拉米苏）
```java
@Factory(
    id = "Margherita",
    type = Meal.class
)
public class MargheritaPizza implements Meal {
  @Override public float getPrice() {
    return 6f;
  }
}

@Factory(
    id = "Tiramisu",
    type = Meal.class
)
public class Tiramisu implements Meal {
  @Override public float getPrice() {
    return 4.5f;
  }
}
```
3、我们希望自动生成这样一个Factory工厂类及工厂方法，能够通过产品id生产对应的产品：
```java
public class MealFactory {

  public Meal create(String id) {
    if (id == null) {
      throw new IllegalArgumentException("id is null!");
    }

    if ("Tiramisu".equals(id)) {
      return new Tiramisu();
    }

    if ("Margherita".equals(id)) {
      return new MargheritaPizza();
    }

    throw new IllegalArgumentException("Unknown id = " + id);
  }
}
```
有了这个MealFactory，我们就可以开一家披萨店了（虽然目前只能提供2种食物，但是后续还可以加)
```java
public class PizzaStore {

  private MealFactory factory = new MealFactory();

  public Meal order(String mealName) {
    return factory.create(mealName);
  }

  public static void main(String[] args) throws IOException {
    PizzaStore pizzaStore = new PizzaStore();
    Meal meal = pizzaStore.order(readConsole());
    System.out.println("Bill: $" + meal.getPrice());
  }
}
```
4、为了实现MealFactory类自动生成，我们使用注解处理器来处理。定义一个注解处理器叫 FactoryProcessor。
```java
@AutoService(Processor.class)//google的注解
public class FactoryProcessor extends AbstractProcessor {

    private Types typeUtils;
    private Elements elementUtils;
    private Filer filer;
    private Messager messager;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        /**
         在init()中我们获得如下引用：
         TypeUtils：一个用来处理TypeMirror的工具类；
         ElementUtils：一个用来处理Element的工具类；
         Filer：使用Filer你可以创建文件；
         Messager：可以用来打印日志；
         */
        typeUtils = processingEnv.getTypeUtils();
        elementUtils = processingEnv.getElementUtils();
        filer = processingEnv.getFiler();
        messager = processingEnv.getMessager();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotations = new LinkedHashSet<String>();
        annotations.add(Factory.class.getCanonicalName());//只处理@Factory注解
        return annotations;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
      ...  
      return true;
    }
}
```

## Element是什么？

在注解处理过程中，我们扫描所有的Java源文件。源代码的每一个部分都是一个特定类型的Element。换句话说：Element代表程序的元素，例如包、类或者方法。每个Element代表一个静态的、语言级别的构件。在下面的例子中，我们通过注释来说明这个：
```java
package com.example;    // PackageElement
public class Foo {      // TypeElement
    private int a;      // VariableElement
    private Foo other;  // VariableElement
    public Foo () {}    // ExecuteableElement
    public void setA (  // ExecuteableElement
            int newA  // VariableElement
            ) {}
}
```
Element是抽象的结点，就像XML解释器一样，有一些类似DOM的元素。你可以从一个元素导航到它的父或者子元素上。

举例来说，假如你有一个 Foo 类的TypeElement元素，你可以遍历它的孩子，如下：
```java
TypeElement fooClass = ... ;  
for (Element e : fooClass.getEnclosedElements()){ // iterate over children  
    Element parent = e.getEnclosingElement();  // parent == fooClass
}
```

## TypeElement

TypeElement代表的是源代码中的类型元素，但是TypeElement并不包含类本身的信息，你可以从TypeElement中获取类的名字，但是你获取不到类的信息，例如它的父类。这种信息需要通过TypeMirror获取，你可以通过调用elements.asType()获取元素的TypeMirror。

5、现在FactoryProcessor剩下一个最重要的方法没有实现，就是process方法，你所做的大部分工作应该从这个方法开始。
第一步工作先扫描@Factory注解
```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    // 遍历所有被注解了@Factory的元素
    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {
      // 检查被注解为@Factory的元素是否是一个类
      if (annotatedElement.getKind() != ElementKind.CLASS) {
        //打印错误信息，非关键性错误最好不要抛异常
            error(annotatedElement, "Only classes can be annotated with @%s",
                  Factory.class.getSimpleName());
        continue;
      }
      ...
    }
}

private void error(Element e, String msg, Object... args) {  
    messager.printMessage(
        Diagnostic.Kind.ERROR,
        String.format(msg, args),
        e);
}
```
判断是不是类，能用! (annotatedElement instanceof TypeElement) 吗？这是错误的，因为接口（interface）类型也是TypeElement。所以在注解处理器中，我们要避免使用instanceof，而是配合TypeMirror使用ElementKind或者TypeKind。

现在我们已经找到了这些被注解在类上的@Factory注解对象(TypeElement)了。

接下来我们要解析注解里的参数了，我们新建一个类FactoryAnnotatedClass来记录解析后的参数，相关的解析工作放在构造函数中进行：
```java
public class FactoryAnnotatedClass {

    private TypeElement annotatedClassElement;
    private String qualifiedSuperClassName;
    private String simpleTypeName;
    private String id;

    public FactoryAnnotatedClass(TypeElement classElement) throws IllegalArgumentException {
        this.annotatedClassElement = classElement;
        //获取类的Factory注解
        Factory annotation = classElement.getAnnotation(Factory.class);
        //从注解获取id参数
        id = annotation.id();

        if (StringUtils.isEmpty(id)) {
            throw new IllegalArgumentException(
                    String.format("id() in @%s for class %s is null or empty! that's not allowed",
                            Factory.class.getSimpleName(), classElement.getQualifiedName().toString()));
        }

        //获取type参数
        try {
            Class<?> clazz = annotation.type();//xxx.yyy.Meal.class
            qualifiedSuperClassName = clazz.getCanonicalName();//xxx.yyy.Meal
            simpleTypeName = clazz.getSimpleName();//Meal
        } catch (MirroredTypeException mte) {
            /**
             annotation.type()返回Class对象，可能发生MirroredTypeException异常。
             那什么时候情况下会发生呢？
             答：因为注解处理是在编译Java源代码之前。我们需要考虑如下两种情况：
             1、这个类已经被编译，例如第三方.jar中的类，已经包含已编译的被@Factory注解.class文件，这时不会发生异常。
             2、这个类还未被编译，直接获取Class会抛出MirroredTypeException异常，单幸运的是，
             MirroredTypeException包含一个TypeMirror，它表示我们未编译类。
             因为我们已经知道它必定是一个类类型（我们已经在前面检查过），我们可以直接强制转换为DeclaredType，
             然后读取TypeElement来获取合法的名字
             */
            DeclaredType classTypeMirror = (DeclaredType) mte.getTypeMirror();
            TypeElement classTypeElement = (TypeElement) classTypeMirror.asElement();
            qualifiedSuperClassName = classTypeElement.getQualifiedName().toString();
            simpleTypeName = classTypeElement.getSimpleName().toString();
        }
    }

    /**
     * 获取在{@link Factory#id()}中指定的id
     * return the id
     */
    public String getId() {
        return id;
    }

    /**
     * 获取在{@link Factory#type()}指定的类型合法全名
     *
     * @return qualified name
     */
    public String getQualifiedFactoryGroupName() {
        return qualifiedSuperClassName;
    }


    /**
     * 获取在 {@link Factory#type()} 中指定的类型的简单名字
     *
     * @return qualified name
     */
    public String getSimpleFactoryGroupName() {
        return simpleTypeName;
    }

    /**
     * 获取被@Factory注解的原始元素
     */
    public TypeElement getTypeElement() {
        return annotatedClassElement;
    }
}
```

现在再建立一个数据结构`FactoryGroupedClasses`，将`FactoryAnnotatedClass`按Map结构组织起来，也就是做一个`id` --> `FactoryAnnotatedClass`的映射关系：

比如：

    "Tiramisu" ->  FactoryAnnotatedClass@1234
    "Margherita" ->  FactoryAnnotatedClass@4321

```java
public class FactoryGroupedClasses {

  private String qualifiedClassName;

  private Map<String, FactoryAnnotatedClass> itemsMap = new LinkedHashMap<String, FactoryAnnotatedClass>();

  public FactoryGroupedClasses(String qualifiedClassName) {
    this.qualifiedClassName = qualifiedClassName;
  }

  public void add(FactoryAnnotatedClass toInsert) throws IdAlreadyUsedException {
    FactoryAnnotatedClass existing = itemsMap.get(toInsert.getId());
    if (existing != null) {
      throw new IdAlreadyUsedException(existing);
    }
    itemsMap.put(toInsert.getId(), toInsert);
  }

  public void generateCode(Elements elementUtils, Filer filer) throws IOException {
    ...
  }
}
```    
现在我们来把process方法补充完，因为Factory注解里的type可能不止有Meal这一种，所以我们可以建一个map结构（factoryClasses）来归类，比如：

    "Meal" --> FactoryGroupedClasses@5678
    "Drink" --> FactoryGroupedClasses@8765

```java
private Map<String, FactoryGroupedClasses> factoryClasses = new LinkedHashMap<String, FactoryGroupedClasses>();

@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {
            ...
            // 因为我们已经知道它是ElementKind.CLASS类型，所以可以直接强制转换
            TypeElement typeElement = (TypeElement) annotatedElement;
            try {
                FactoryAnnotatedClass annotatedClass = new FactoryAnnotatedClass(typeElement); // may throws IllegalArgumentException
                if (!isValidClass(annotatedClass)) {//额外的其他检查
                    continue;
                }

                // 所有检查都没有问题，所以可以添加了
                FactoryGroupedClasses factoryClass = factoryClasses.get(annotatedClass.getQualifiedFactoryGroupName());
                if (factoryClass == null) {
                    String qualifiedGroupName = annotatedClass.getQualifiedFactoryGroupName();
                    factoryClass = new FactoryGroupedClasses(qualifiedGroupName);
                    factoryClasses.put(qualifiedGroupName, factoryClass);
                }
                // 如果和其他的@Factory标注的类的id相同冲突，抛出IdAlreadyUsedException异常
                factoryClass.add(annotatedClass);
            } catch (IllegalArgumentException e) {
                // @Factory.id()为空 --> 打印错误信息
                error(typeElement, e.getMessage());
                continue;
            } catch (IdAlreadyUsedException e) {
                FactoryAnnotatedClass existing = e.getExisting();
                // 已经存在
                error(annotatedElement,
                        "Conflict: The class %s is annotated with @%s with id ='%s' but %s already uses the same id",
                        typeElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
                        existing.getTypeElement().getQualifiedName().toString());
                continue;
            }

            try {
                for (FactoryGroupedClasses factoryClass : factoryClasses.values()) {
                    //生成代码
                    factoryClass.generateCode(elementUtils, filer);
                }
            } catch (IOException e) {
                error(null, e.getMessage());
            }
          	//务必清空成员变量缓存factoryClasses
        		factoryClasses.clear();
            return true;
        }
    }

    private boolean isValidClass(FactoryAnnotatedClass item) {
        // 转换为TypeElement, 含有更多特定的方法
        TypeElement classElement = item.getTypeElement();
        // 检查是否是public的类
        if (!classElement.getModifiers().contains(Modifier.PUBLIC)) {
            error(classElement, "The class %s is not public.", classElement.getQualifiedName().toString());
            return false;
        }

        // 检查是否是抽象的
        if (classElement.getModifiers().contains(Modifier.ABSTRACT)) {
            error(classElement, "The class %s is abstract. You can't annotate abstract classes with @%",
                    classElement.getQualifiedName().toString(), Factory.class.getSimpleName());
            return false;
        }

        // 检查继承关系: 必须是@Factory.type()指定的类型子类，比如：Tiramisu必须是Meal类型的
        TypeElement superClassElement =
                elementUtils.getTypeElement(item.getQualifiedFactoryGroupName());//Meal
        if (superClassElement.getKind() == ElementKind.INTERFACE) {//如果Meal是个接口
            // 检查接口是否实现了
            if (!classElement.getInterfaces().contains(superClassElement.asType())) {//Tiramisu的接口中是否包含Meal接口
                error(classElement, "The class %s annotated with @%s must implement the interface %s",
                        classElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
                        item.getQualifiedFactoryGroupName());
                return false;
            }
        } else {//如果Meal是个类
            // 循环检查父类
            TypeElement currentClass = classElement;
            while (true) {
                TypeMirror superClassType = currentClass.getSuperclass();//父类

                if (superClassType.getKind() == TypeKind.NONE) {
                    // 到达了基本类型(java.lang.Object), 所以退出
                    error(classElement, "The class %s annotated with @%s must inherit from %s",
                            classElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
                            item.getQualifiedFactoryGroupName());
                    return false;
                }

                if (superClassType.toString().equals(item.getQualifiedFactoryGroupName())) {
                    // 找到了要求的父类
                    break;
                }

                // 在继承树上继续向上搜寻
                currentClass = (TypeElement) typeUtils.asElement(superClassType);
            }
        }

        // 检查是否提供了默认公开构造函数，因为我们需要的代码中要使用默认构造函数，比如new Tiramisu()
        for (Element enclosed : classElement.getEnclosedElements()) {
            if (enclosed.getKind() == ElementKind.CONSTRUCTOR) {
                ExecutableElement constructorElement = (ExecutableElement) enclosed;
                if (constructorElement.getParameters().size() == 0 && constructorElement.getModifiers()
                        .contains(Modifier.PUBLIC)) {
                    // 找到了默认构造函数
                    return true;
                }
            }
        }

        // 没有找到默认构造函数
        error(classElement, "The class %s must provide an public empty default constructor",
                classElement.getQualifiedName().toString());
        return false;
}
```

现在解析工作结束了，剩下的就是生成对应的java代码了，我们来实现FactoryGroupedClassesFactoryGroupedClasses中遗留的generateCode方法。

这里使用`javapoet`动态生成java源文件：
```java
private static final String SUFFIX = "Factory";

public void generateCode(Elements elementUtils, Filer filer) throws IOException {
     		TypeElement superClassName = elementUtils.getTypeElement(qualifiedClassName);//Meal
        String factoryClassName = superClassName.getSimpleName() + SUFFIX;//"MealFactory"

        MethodSpec.Builder methodSpecBuilder = MethodSpec.methodBuilder("create")//public Meal create(String id)
                .addModifiers(Modifier.PUBLIC)
                .addParameter(ParameterSpec.builder(String.class, "id").build())
                .returns(TypeName.get(superClassName.asType()));

        CodeBlock.Builder toStringCodeBuilder = CodeBlock.builder();
        toStringCodeBuilder.beginControlFlow("if (id == null)");
        toStringCodeBuilder.add(CodeBlock.of("throw new IllegalArgumentException(\"id is null!\");\n"));
        toStringCodeBuilder.endControlFlow();
        methodSpecBuilder.addCode(toStringCodeBuilder.build());

        for (FactoryAnnotatedClass item : itemsMap.values()) {
            toStringCodeBuilder = CodeBlock.builder();
            toStringCodeBuilder.beginControlFlow("if ($S.equals(id))", item.getId());
            toStringCodeBuilder.add(CodeBlock.of("return new $T();\n", item.getTypeElement()));
            toStringCodeBuilder.endControlFlow();
            methodSpecBuilder.addCode(toStringCodeBuilder.build());
        }
        methodSpecBuilder.addCode(CodeBlock.of("throw new IllegalArgumentException(\"Unknown id = \" + id);\n"));

        TypeSpec typeSpec = TypeSpec.classBuilder(factoryClassName)
                .addModifiers(Modifier.PUBLIC)
                .addMethod(methodSpecBuilder.build())
                .build();

        JavaFile.builder("com.example.annotation", typeSpec)//生成com.example.annotation.MealFactory
                .build().writeTo(filer);
}
```

最后是测试代码：
```java
public class Test {
    public static void main(String[] args) {
        System.out.println(new MealFactory().create("Tiramisu").getPrice());
    }
}
```

完整代码：[https://github.com/qylk/MyAnnotationTest/tree/master](https://github.com/qylk/MyAnnotationTest/tree/master)