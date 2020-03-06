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
createInsertCode方法是插桩的核心方法。首先是 prepareMethodParameters 用来准备参数数组。

```java
public static void createInsertCode(GeneratorAdapter mv, String className, List<Type> args, Type returnType, boolean isStatic, int methodId) {
        //准备参数
        prepareMethodParameters(mv, className, args, returnType, isStatic, methodId);
        //...
    }
```
参数数组是指什么呢？再看下插桩里的方法
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

```java
    private static void prepareMethodParameters(GeneratorAdapter mv, String className, List<Type> args, Type returnType, boolean isStatic, int methodId) {
        //第一个参数：new Object[]{...};,如果方法没有参数直接传入new Object[0]
        if (args.size() == 0) {
            mv.visitInsn(Opcodes.ICONST_0);//将常量0载入栈
            mv.visitTypeInsn(Opcodes.ANEWARRAY, "java/lang/Object");//new Object[0]入栈
        } else {
            createObjectArray(mv, args, isStatic);//创建new Object[]{...}入栈
        }

        //第二个参数：this,如果方法是static的话就直接传入null
        if (isStatic) {
            mv.visitInsn(Opcodes.ACONST_NULL);//null入栈
        } else {
            mv.visitVarInsn(Opcodes.ALOAD, 0);//this入栈
        }

        //第三个参数：changeQuickRedirect
        mv.visitFieldInsn(Opcodes.GETSTATIC,//changeQuickRedirect入栈
                className,
                REDIRECTFIELD_NAME,
                REDIRECTCLASSNAME);

        //第四个参数：false,标志是否为static
        mv.visitInsn(isStatic ? Opcodes.ICONST_1 : Opcodes.ICONST_0);//isStatic入栈
        //第五个参数：
        mv.push(methodId);//methodId入栈
        //第六个参数：参数class数组
        createClassArray(mv, args);
        //第七个参数：返回值类型class
        createReturnClass(mv, returnType);
    }
```

```java
   private static void createObjectArray(MethodVisitor mv, List<Type> paramsTypeClass, boolean isStatic) {
        //Opcodes.ICONST_0 ~ Opcodes.ICONST_5 这个指令范围
        int argsCount = paramsTypeClass.size();
        //声明 Object[argsCount];
        if (argsCount >= 6) {
            mv.visitIntInsn(Opcodes.BIPUSH, argsCount);
        } else {
            mv.visitInsn(Opcodes.ICONST_0 + argsCount);
        }
        mv.visitTypeInsn(Opcodes.ANEWARRAY, "java/lang/Object");//new Object[argsCount]，栈：[array]

        //如果是static方法，没有this隐含参数
        int loadIndex = (isStatic ? 0 : 1);

        //填充数组数据
        for (int i = 0; i < argsCount; i++) {
            mv.visitInsn(Opcodes.DUP);//栈:[array,array]
            if (i <= 5) {
                mv.visitInsn(Opcodes.ICONST_0 + i);//位置i入栈, 栈:[array,array,i]
            } else {
                mv.visitIntInsn(Opcodes.BIPUSH, i);
            }
            if (!createPrimateTypeObj(mv, loadIndex, paramsTypeClass.get(i).getDescriptor())) {
                mv.visitVarInsn(Opcodes.ALOAD, loadIndex);//[array,array,i,args]
                mv.visitInsn(Opcodes.AASTORE);//[array]
            }
            loadIndex++;
        }
    }

    //将基本类型转换为包装类型
    private static boolean createPrimateTypeObj(MethodVisitor mv, int argsPostion, String typeS) {
        if ("Z".equals(typeS)) {
            createBooleanObj(mv, argsPostion);
            return true;
        }
        if ("B".equals(typeS)) {
            createBooleanObj(mv, argsPostion);
            return true;
        }
        //....等等等
        return false;
    }

    private static void createBooleanObj(MethodVisitor mv, int argsPosition) {
        mv.visitTypeInsn(Opcodes.NEW, "java/lang/Byte");//栈：[array,array,i,newByte]
        mv.visitInsn(Opcodes.DUP);//栈：[array,array,i,newByte,newByte]
        mv.visitVarInsn(Opcodes.ILOAD, argsPosition);//参数值入栈，栈：[array,array,i,newByte,newByte,args]
        mv.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/Byte", "<init>", "(B)V");//调用构造函数，栈:[array,array,i,newByteArgs]
        mv.visitInsn(Opcodes.AASTORE);//栈：[array]
    }
```
prepareMethodParameters 就这些，继续往下看：
```java
        prepareMethodParameters(mv, className, args, returnType, isStatic, methodId);
        
        //开始调用，PatchProxy.proxy(...)
        mv.visitMethodInsn(Opcodes.INVOKESTATIC,
                PROXYCLASSNAME,
                "proxy",
                "([Ljava/lang/Object;Ljava/lang/Object;" + REDIRECTCLASSNAME + "ZI[Ljava/lang/Class;Ljava/lang/Class;)Lcom/meituan/robust/PatchProxyResult;",
                false);

        int local = mv.newLocal(Type.getType("Lcom/meituan/robust/PatchProxyResult;"));
        mv.storeLocal(local);
        mv.loadLocal(local);

        //result.isSupported
        mv.visitFieldInsn(Opcodes.GETFIELD, "com/meituan/robust/PatchProxyResult", "isSupported", "Z");

        // if isSupported
        Label l1 = new Label();
        mv.visitJumpInsn(Opcodes.IFEQ, l1);//如果isNotSupported，跳到l1执行

        //判断是否有返回值，代码不同
        if ("V".equals(returnType.getDescriptor())) {
            mv.visitInsn(Opcodes.RETURN);
        } else {
            mv.loadLocal(local);
            //result.result
            mv.visitFieldInsn(Opcodes.GETFIELD, "com/meituan/robust/PatchProxyResult", "result", "Ljava/lang/Object;");
            //强制转化类型
            if (!castPrimateToObj(mv, returnType.getDescriptor())) {
                //这里需要注意，如果是数组类型的直接使用即可，如果非数组类型，就得去除前缀L,还有最终是没有结束符;
                //比如：Ljava/lang/String; ==> java/lang/String
                String newTypeStr = null;
                int len = returnType.getDescriptor().length();
                if (returnType.getDescriptor().startsWith("[")) {
                    newTypeStr = returnType.getDescriptor().substring(0, len);
                } else {
                    newTypeStr = returnType.getDescriptor().substring(1, len - 1);
                }
                mv.visitTypeInsn(Opcodes.CHECKCAST, newTypeStr);
            }

            //这里还需要做返回类型不同返回指令也不同
            mv.visitInsn(getReturnTypeCode(returnType.getDescriptor()));
        }

        mv.visitLabel(l1);
    }

    private static boolean castPrimateToObj(MethodVisitor mv, String typeS) {
        if ("Z".equals(typeS)) {
            mv.visitTypeInsn(Opcodes.CHECKCAST, "java/lang/Boolean");//强制转化类型
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/Boolean", "booleanValue", "()Z");
            return true;
        }
        //。。。等等等
    }
```
方法插桩逻辑基本就以上这些。

## patch逻辑
再回顾下方法中的桩：
```java
public long patchedMethod() {
    PatchProxyResult patchProxyResult = PatchProxy.proxy(new Object[0], this, changeQuickRedirect, false,/*methodNumber*/ 32, new Class[0], long.class);
    if (patchProxyResult.isSupported) {
        return ((Long) patchProxyResult.result).longValue();
    }

    //原有逻辑
}
```
调用PatchProxy.proxy，并判断返回值中的isSupported来决定是否已经被打了补丁，否则走原有的逻辑。
```java
  public static PatchProxyResult proxy(Object[] paramsArray, Object current, ChangeQuickRedirect changeQuickRedirect, boolean isStatic, int methodNumber, Class[] paramsClassTypes, Class returnType) {
        PatchProxyResult patchProxyResult = new PatchProxyResult();
        if (PatchProxy.isSupport(paramsArray, current, changeQuickRedirect, isStatic, methodNumber, paramsClassTypes, returnType)) {
            patchProxyResult.isSupported = true;
            patchProxyResult.result = PatchProxy.accessDispatch(paramsArray, current, changeQuickRedirect, isStatic, methodNumber, paramsClassTypes, returnType);
        }
        return patchProxyResult;
    }
```
先是调用isSupport判断是否有补丁存在，如果有，就执行accessDispatch来执行补丁中的逻辑，并返回修复后的方法返回值。
```java
public static boolean isSupport(Object[] paramsArray, Object current, ChangeQuickRedirect changeQuickRedirect, boolean isStatic, int methodNumber, Class[] paramsClassTypes, Class returnType) {
        //Robust补丁优先执行，其他功能靠后
        if (changeQuickRedirect == null) {
            //不执行补丁，轮询其他监听者
            if (registerExtensionList == null || registerExtensionList.isEmpty()) {
                return false;
            }
            for (RobustExtension robustExtension : registerExtensionList) {
                if (robustExtension.isSupport(new RobustArguments(paramsArray, current, isStatic, methodNumber, paramsClassTypes, returnType))) {
                    robustExtensionThreadLocal.set(robustExtension);
                    return true;
                }
            }
            return false;
        }

        String classMethod = getClassMethod(isStatic, methodNumber);
        if (TextUtils.isEmpty(classMethod)) {
            return false;
        }
        Object[] objects = getObjects(paramsArray, current, isStatic);
        try {
            return changeQuickRedirect.isSupport(classMethod, objects);
        } catch (Throwable t) {
            return false;
        }
    }
```
这里当changeQuickRedirect==null时，通过遍历 List<RobustExtension> 来判断是否有补丁。而当changeQuickRedirect != null时，通过changeQuickRedirect.isSupport来判断，这个一般都返回true。
注意isSupport传递的参数：一个数classMethod，一个是参数object数组。
其中获取classMethod如下，形如："::false:32"的形式。
```java
private static String getClassMethod(boolean isStatic, int methodNumber) {
        String classMethod = "";
        try {
            String methodName = "";
            String className = "";
            classMethod = className + ":" + methodName + ":" + isStatic + ":" + methodNumber;
        } catch (Exception e) {
        }
        return classMethod;
    }
```
changeQuickRedirect是接口，其实现类在真正的patch包内，先不看。
判断完是否有补丁之后，就是执行补丁了。方法从accessDispatch开始。
```java
    public static Object accessDispatch(Object[] paramsArray, Object current, ChangeQuickRedirect changeQuickRedirect, boolean isStatic, int methodNumber, Class[] paramsClassTypes, Class returnType) {
        //和isSupport逻辑是一致的。
        if (changeQuickRedirect == null) {
            RobustExtension robustExtension = robustExtensionThreadLocal.get();
            robustExtensionThreadLocal.remove();
            if (robustExtension != null) {
                notify(robustExtension.describeSelfFunction());
                return robustExtension.accessDispatch(new RobustArguments(paramsArray, current, isStatic, methodNumber, paramsClassTypes, returnType));
            }
            return null;
        }


        String classMethod = getClassMethod(isStatic, methodNumber);
        if (TextUtils.isEmpty(classMethod)) {
            return null;
        }
        notify(Constants.PATCH_EXECUTE);
        Object[] objects = getObjects(paramsArray, current, isStatic);
        return changeQuickRedirect.accessDispatch(classMethod, objects);
    }
```
最后调用changeQuickRedirect.accessDispatch来执行patch代码，这个逻辑也在patch包。
所以总结下来，patch包里补丁要实现changeQuickRedirect的两个方法，isSupport和accessDispatch。

### 补丁包
再看一个补丁包包含哪些东西？
补丁包是针对补丁方法打出来的针对性的几个类的集合，仍以dex形式提供。
补丁结构：每个补丁包含以下三个类。
1. PatchesInfoImpl：
补丁包描述，可以获取所有补丁对象；每个对象包含被修复类名及该类对应的补丁类。
```java
public class PatchesInfoImpl implements PatchesInfo
 {
  public List getPatchedClassesInfo()
   {
     ArrayList localArrayList = new ArrayList();
     localArrayList.add(new  PatchedClassInfo("com.test.android.robustdemo.MainActivity", "xxx.robust.patch.MainActivityPatchControl"));
     com.meituan.robust.utils.EnhancedRobustUtils.isThrowable = false;
     return localArrayList;
   }
  }
```
2. PatchControl:
补丁类，具备判断方法是否执行补丁逻辑，及补丁方法的调度。
```java
public class MainActivityPatchControl implements ChangeQuickRedirect {
  ...
  //1.方法是否支持热修
  public boolean isSupport(String paramString, Object[] paramArrayOfObject)
  {
    //从classMethod取出methodNumber
    String str = paramString.split(":")[3];
    //通过methodNumber来匹配
    return ":2:".contains(":" + str + ":");
  }

  //2.调用补丁的热修逻辑
  public Object accessDispatch(String paramString, Object[] paramArrayOfObject) {
      //。。。补丁方法调度
  }
```
3. Patch：
具体补丁方法的实现。该类中包含被修复类中需要热修的方法。
```java
public class MainActivityPatch {
    MainActivity originClass;
    public MainActivityPatch(Object paramObject) {
        this.originClass = ((MainActivity)paramObject);
    }

    //热修的方法具体实现
    private String getShowText() {
        Object localObject = getRealParameter(new Object[] { "Error Text" });
        localObject = (String)EnhancedRobustUtils.invokeReflectConstruct("java.lang.String", (Object[])localObject, new Class[] { String.class });
        localObject = getRealParameter(new Object[] { "Fixed Text" });
        return (String)EnhancedRobustUtils.invokeReflectConstruct("java.lang.String", (Object[])localObject, new Class[] { String.class });
     }
}
```

### 加载patch
pacth的拉取和加载是由PatchExecutor来执行的，其本身是一个线程。
```java
public class PatchExecutor extends Thread {
    protected Context context;
    protected PatchManipulate patchManipulate;
    protected RobustCallBack robustCallBack;

    public PatchExecutor(Context context, PatchManipulate patchManipulate, RobustCallBack robustCallBack) {
        this.context = context.getApplicationContext();
        this.patchManipulate = patchManipulate;
        this.robustCallBack = robustCallBack;
    }

    @Override
    public void run() {
        try {
            //拉取补丁列表
            List<Patch> patches = fetchPatchList();
            //应用补丁列表
            applyPatchList(patches);
        } catch (Throwable t) {
            Log.e("robust", "PatchExecutor run", t);
            robustCallBack.exceptionNotify(t, "class:PatchExecutor,method:run,line:36");
        }
    }
    //...
}
```
其中fetchPatchList是从服务端获取该apk对应的patch列表，得到List\<Patch\>。
```java
public class Patch implements Cloneable {
    private String patchesInfoImplClassFullName;
    /**
     * 补丁名称
     */
    private String name;
    /**
     * 补丁的下载url
     */
    private String url;
    /**
     * 补丁本地保存路径
     */
    private String localPath;
    private String tempPath;
    /**
     * 补丁md5值
     */
    private String md5;
    /**
     * app hash值,避免应用内升级导致低版本app的补丁应用到了高版本app上
     */
    private String appHash;

    /**
     * 补丁是否已经applied success
     */
    private boolean isAppliedSuccess;
}
```
Patch类只记录了Patch相关的一些基本信息，如patch路径url、path、name等。  
然后是applyPatchList。
```java
    protected void applyPatchList(List<Patch> patches) {
        if (null == patches || patches.isEmpty()) {
            return;
        }
        for (Patch p : patches) {
            if (p.isAppliedSuccess()) {
                continue;
            }
            if (patchManipulate.ensurePatchExist(p)) {
                boolean currentPatchResult = false;
                try {
                    currentPatchResult = patch(context, p);
                } catch (Throwable t) {
                }
                if (currentPatchResult) {
                    //设置patch 状态为成功
                    p.setAppliedSuccess(true);
                    //统计PATCH成功率 PATCH成功
                    robustCallBack.onPatchApplied(true, p);
                } else {
                    //统计PATCH成功率 PATCH失败
                    robustCallBack.onPatchApplied(false, p);
                }
            }
        }
    }
```

```java
protected boolean patch(Context context, Patch patch) {
        //校验patch
        if (!patchManipulate.verifyPatch(context, patch)) {
            return false;
        }

        ClassLoader classLoader = null;

        try {
            File dexOutputDir = getPatchCacheDirPath(context, patch.getName() + patch.getMd5());
            //建立patch的ClassLoader
            classLoader = new DexClassLoader(patch.getTempPath(), dexOutputDir.getAbsolutePath(),
                    null, PatchExecutor.class.getClassLoader());
        } catch (Throwable throwable) {
        }

        Class patchClass, sourceClass;
        Class patchesInfoClass;
        PatchesInfo patchesInfo = null;
        try {
            //加载类PatchesInfoImplClass
            patchesInfoClass = classLoader.loadClass(patch.getPatchesInfoImplClassFullName());
            patchesInfo = (PatchesInfo) patchesInfoClass.newInstance();
        } catch (Throwable t) {
        }
        
        //从patchesInfoClass中获得PatchedClassInfo列表，参考上面patch包的第一个类。
        List<PatchedClassInfo> patchedClasses = patchesInfo.getPatchedClassesInfo();
        boolean isClassNotFoundException = false;
        for (PatchedClassInfo patchedClassInfo : patchedClasses) {
            String patchedClassName = patchedClassInfo.patchedClassName;//如：MainActivity
            String patchClassName = patchedClassInfo.patchClassName;//如：MainActivityPatchControl
            try {
                try {
                    //加载被修复类，如MainActivity
                    sourceClass = classLoader.loadClass(patchedClassName.trim());
                } catch (ClassNotFoundException e) {
                    isClassNotFoundException = true;
                    continue;
                }

                Field[] fields = sourceClass.getDeclaredFields();
                Field changeQuickRedirectField = null;
                //遍历找出changeQuickRedirectField字段
                for (Field field : fields) {
                    if (TextUtils.equals(field.getType().getCanonicalName(), ChangeQuickRedirect.class.getCanonicalName()) && TextUtils.equals(field.getDeclaringClass().getCanonicalName(), sourceClass.getCanonicalName())) {
                        changeQuickRedirectField = field;
                        break;
                    }
                }
                if (changeQuickRedirectField == null) {
                    continue;
                }
                try {
                    //加载xxxPatchControl类，如MainActivityPatchControl
                    patchClass = classLoader.loadClass(patchClassName);
                    Object patchObject = patchClass.newInstance();
                    //设置到changeQuickRedirectField字段上，完成补丁对象注入
                    changeQuickRedirectField.setAccessible(true);
                    changeQuickRedirectField.set(null, patchObject);
                } catch (Throwable t) {
                }
            } catch (Throwable t) {
            }
        }
        if (isClassNotFoundException) {
            return false;
        }
        return true;
    }
```

## 自动化补丁
自动化补丁具体实现在auto-patch-plugin Moudle中，也是利用了Transform API自动生成。
简单说就是首先将包含修复完代码的类复制一份，然后将方法里的字段、方法调用都使用反射方式调用。为什么用反射呢？因为patch里只包含要修复的代码，调用非修复方法内的代码正常是调用不到的，所以使用反射。
//todo 


参考：<http://w4lle.com/2017/03/31/robust-0/index.html>