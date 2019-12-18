//https://www.jianshu.com/p/2d7b42cbfa70

# Robust

## 修复原理
1. 在每个类中插入一个静态变量，在每个方法前插入跳转控制逻辑
```java
public long getIndex() {
        System.out.println("patch here");
        return 100;
}
```
以上代码经过Robust框架注入后会被处理成：
```java
public static ChangeQuickRedirect changeQuickRedirect;

public long getIndex() {
    PatchProxyResult patchProxyResult = PatchProxy.proxy(new Object[0], this, changeQuickRedirect, false, 32, new Class[0], long.class);
    if (patchProxyResult.isSupported) {
        return ((Long) patchProxyResult.result).longValue();
    }

    System.out.println("patch here");
    return 100L;
}
```
当有补丁的时候 `changeQuickRedirect` 的值就不再是空，所以执行到需要热修的方法时就会走到补丁的方法实现而不是原逻辑达到修复目的。该原理同Instant Run类似，都是采用插桩的方式通过运行时跳转来改变代码执行路径达到修复的目的。

Robust实现原理主要包含三部分：基础包插桩、补丁加载及自动化补丁。
### 基础包插桩
利用gradle-plugin注册Transform在编译时修改Class文件进行插桩。
```groovy
class RobustTransform extends Transform implements Plugin<Project> {
    @Override
    void apply(Project target) {
        //注册Transform
        robust = new XmlSlurper().parse(new File("${project.projectDir}/${Constants.ROBUST_XML}"))
        project.android.registerTransform(this)
    }

    @Override
    void transform(Context context, Collection<TransformInput> inputs, Collection<TransformInput> referencedInputs, TransformOutputProvider outputProvider, boolean isIncremental) throws IOException, TransformException, InterruptedException {
        //...
        //修改class对象：支持通过 Asm 或者 JavaAssist 操作修改class对象。
        if(useASM){
            insertcodeStrategy=new AsmInsertImpl(hotfixPackageList,hotfixMethodList,exceptPackageList,exceptMethodList,isHotfixMethodLevel,isExceptMethodLevel);
        } else {
            insertcodeStrategy=new JavaAssistInsertImpl(hotfixPackageList,hotfixMethodList,exceptPackageList,exceptMethodList,isHotfixMethodLevel,isExceptMethodLevel);
        }
        //执行代码插入
        insertcodeStrategy.insertCode(box, jarFile);
        //...
    }
}
```

默认采用Asm，看下AsmInsertImpl的insertCode方法。
```groovy
 @Override
    protected void insertCode(List<CtClass> box, File jarFile) throws IOException, CannotCompileException {
        ZipOutputStream outStream = new JarOutputStream(new FileOutputStream(jarFile));
        //get every class in the box ,ready to insert code
        for (CtClass ctClass : box) {
            //change modifier to public ,so all the class in the apk will be public ,you will be able to access it in the patch
            ctClass.setModifiers(AccessFlag.setPublic(ctClass.getModifiers()));
            if (isNeedInsertClass(ctClass.getName()) && !(ctClass.isInterface() || ctClass.getDeclaredMethods().length < 1)) {
                //only insert code into specific classes
                zipFile(transformCode(ctClass.toBytecode(), ctClass.getName().replaceAll("\\.", "/")), outStream, ctClass.getName().replaceAll("\\.", "/") + ".class");
            } else {
                zipFile(ctClass.toBytecode(), outStream, ctClass.getName().replaceAll("\\.", "/") + ".class");
            }
        }
        outStream.close();
    }
```
将Class对象修改后打入jar包，可以看到将所有方法都改成了public，isNeedInsertClass()用于判断白名单和黑名单，对于接口类和空类跳过修改。重点看transformCode方法。
```groovy
public byte[] transformCode(byte[] b1, String className) throws IOException {
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);
        ClassReader cr = new ClassReader(b1);
        ClassNode classNode = new ClassNode();
        Map<String, Boolean> methodInstructionTypeMap = new HashMap<>();
        cr.accept(classNode, 0);
        final List<MethodNode> methods = classNode.methods;
        //收集method
        for (MethodNode m : methods) {
            InsnList inList = m.instructions;
            boolean isMethodInvoke = false;
            for (int i = 0; i < inList.size(); i++) {
                if (inList.get(i).getType() == AbstractInsnNode.METHOD_INSN) {
                    isMethodInvoke = true;
                }
            }
            methodInstructionTypeMap.put(m.name + m.desc, isMethodInvoke);
        }
        
        //ASM插桩
        InsertMethodBodyAdapter insertMethodBodyAdapter = new InsertMethodBodyAdapter(cw, className, methodInstructionTypeMap);
        cr.accept(insertMethodBodyAdapter, ClassReader.EXPAND_FRAMES);
        return cw.toByteArray();
}
```
InsertMethodBodyAdapter 继承自 ClassVisitor ，用于访问class对象，首先在InsertMethodBodyAdapter的构造函数中，使用visitField为类添加了
静态变量changeQuickRedirect。然后是方法插桩，相关的逻辑在visitMethod回调中。
```java
public InsertMethodBodyAdapter(ClassWriter cw, String className, Map<String, Boolean> methodInstructionTypeMap) {
    super(Opcodes.ASM5, cw);
    this.classWriter = cw;
    this.className = className;
    this.methodInstructionTypeMap = methodInstructionTypeMap;
    //插入静态变量changeQuickRedirect
    classWriter.visitField(Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC, Constants.INSERT_FIELD_NAME, Type.getDescriptor(ChangeQuickRedirect.class), null, null);
}

@Override
public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
    //protedted类改成publish类，否则可能无法访问该类的原始方法
    if (isProtect(access)) {
        access = setPublic(access);
    }
    MethodVisitor mv = super.visitMethod(access, name,
            desc, signature, exceptions);
    //方法过滤，白名单+黑名单
    if (!isQualifiedMethod(access, name, desc, methodInstructionTypeMap)) {
        return mv;
    }
    //构造方法签名，主要用来从methodMap文件中找到混淆后的方法名
    StringBuilder parameters = new StringBuilder();
    Type[] types = Type.getArgumentTypes(desc);
    for (Type type : types) {
        parameters.append(type.getClassName()).append(",");
    }
    //remove the last ","
    if (parameters.length() > 0 && parameters.charAt(parameters.length() - 1) == ',') {
        parameters.deleteCharAt(parameters.length() - 1);
    }
    //record method number
    methodMap.put(className.replace('/', '.') + "." + name + "(" + parameters.toString() + ")", insertMethodCount.incrementAndGet())

    //方法插桩
    return new MethodBodyInsertor(mv, className, desc, isStatic(access), String.valueOf(insertMethodCount.get()), name, access);
}
```
MethodBodyInsertor 是一个 MethodVisitor，visitCode是访问方法体内的代码时的回调。
```java
class MethodBodyInsertor extends GeneratorAdapter {
    @Override
    public void visitCode() {
      //insert code here
     RobustAsmUtils.createInsertCode(this, className, paramsTypeClass, returnType, isStatic, Integer.valueOf(methodId));
    }
}
```
createInsertCode方法是插桩的核心方法。首先是prepareMethodParameters用来准备参数数组。

```java
public static void createInsertCode(GeneratorAdapter mv, String className, List<Type> args, Type returnType, boolean isStatic, int methodId) {
        //准备参数
        prepareMethodParameters(mv, className, args, returnType, isStatic, methodId);
        //...
    }
```
参数是指什么呢？再看下插桩的方法
```java
PatchProxy.proxy(new Object[0], this, changeQuickRedirect, false, 32, new Class[0], long.class);
```
就是指PatchProxy.proxy括号内的参数，一共7个。
1. paramsArray：Object[]
    原方法参数列表数组  
2. current：Object
    this参数
3. changeQuickRedirect：ChangeQuickRedirect
    就那个类静态变量
4. isStatic：boolean
    是否是static方法    
5. methodNumber：int
    methodId
6. paramsClassTypes：Class[]
    参数类型数组
7. returnType：Class
    返回值类型
