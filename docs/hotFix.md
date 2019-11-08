![fix](./assets/30.png)

## multidex类加载方案
multidex类加载方案是要求应用重启后，优先加载补丁包中的类，从而达到修复的目的，为什么要求重启呢？因为在java中类是无法卸载的，如果要重新加载类，只能重启jvm，因此这类修复方案的特点是`不能即时生效`，需要冷启动。

我们从ClassLoader类加载的逻辑([ClassLoader](./classloader.md))中可知，在android中，classLoader在第一次加载类的时候其实是按顺序遍历dexElements数组，从每一个DexFile中尝试加载类，如果找到了，下次就不会重复加载了，而是从已加载的类中直接查找。

因此我们可以将补丁类打包成patch.dex，放到dexElement数组的第一个元素，这样虚拟机在加载类的时候会先找到我们补丁包patch.dex中已经修复的类，而有问题的类因为排在后面的dex中，就也没有机会被加载。  
![QFix](./assets/31.png)


### 实践
1、修改需要修复的类
```java
package com.test.fix;

public class Test() {

    public void test(){
        //sout("出bug了");
        fix();
    }

    public void fix(){
        sout("修复了");
    }

}
```

2、编译并找到补丁class，生成补丁包
```bash
##生成jar
jar  cvf class.jar com/test/fix/Test.class
##生成dex文件，在patchDex.jar中
dx --dex --output=patchDex.jar class.jar 
```

3、下发patch到终端进行修复
```java
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        String dexPath = Environment.getExternalStorageDirectory().getAbsolutePath().concat("/patchDex.jar");
        File file = new File(dexPath);
        //如果补丁存在就执行修复
        if (file.exists()) {
            inject(dexPath);
        }
    }

    private void inject(String dexFile) {
        try {
            // 获取主classes的dexElements
            Class<?> cl = Class.forName("dalvik.system.BaseDexClassLoader");
            Object pathList = getField(cl, "pathList", getClassLoader());
            Object baseElements = getField(pathList.getClass(), "dexElements", pathList);


            //使用dexClassLoader加载patch.dex
            String dexopt = getCacheDir().getAbsolutePath();
            DexClassLoader dexClassLoader = new DexClassLoader(dexFile, dexopt, dexopt, getClassLoader());
            //获取patch.dex的dexElements
            Object obj = getField(cl, "pathList", dexClassLoader);
            Object patchDexElements = getField(obj.getClass(), "dexElements", obj);

            //合并两个dexElements, patchDexElements放前面
            Object combineElements = combineArray(patchDexElements, baseElements);

            //将合并后的Element数组重新赋值给app的classLoader
            setField(pathList.getClass(), "dexElements", pathList, combineElements);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
    }

 
    private Object getField(Class<?> cl, String fieldName, Object object) throws NoSuchFieldException, IllegalAccessException {
        Field field = cl.getDeclaredField(fieldName);
        field.setAccessible(true);
        return field.get(object);
    }

  
    private void setField(Class<?> cl, String fieldName, Object object, Object value) throws NoSuchFieldException, IllegalAccessException {
        Field field = cl.getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(object, value);
    }

   
    private Object combineArray(Object firstArr, Object secondArr) {
        int firstLength = Array.getLength(firstArr);
        int secondLength = Array.getLength(secondArr);
        int length = firstLength + secondLength;

        Class<?> componentType = firstArr.getClass().getComponentType();
        Object newArr = Array.newInstance(componentType, length);
        for (int i = 0; i < length; i++) {
            if (i < firstLength) {
                Array.set(newArr, i, Array.get(firstArr, i));
            } else {
                Array.set(newArr, i, Array.get(secondArr, i - firstLength));
            }
        }
        return newArr;
    }

}
```

该方案在android 4.x版本上会出现一个异常：
```bash
java.lang.IllegalAccessError: 
    Class ref in pre-verified class resolved to unexpected implementation
```
原因是：  
1、在apk安装的时候，Dalvik虚拟机会将dex优化成odex后才拿去执行，在这个过程中会对所有class一个校验。  
2、校验方式：假设A该类在它的static方法，private方法，构造函数，override方法中直接引用到B类。如果A类和B类在同一个dex中，那么A类就会被打上`CLASS_ISPREVERIFIED`标记，反之只要在static方法，构造方法，private方法，override方法中直接引用了其他dex中的类，那么这个类就不会被打上`CLASS_ISPREVERIFIED`标记。
3、被打上这个标记的类就不能再引用其他dex中的类，否则就会报上面的错误。

在我们的实践中，Test类和调用Test类的MainActivity(比如)本身是在同一个dex中的，所以MainActivity被打上了`CLASS_ISPREVERIFIED`标记，而我们修复bug的时候却引用了另外一个dex中的Test，所以这里就报错了。

QFix为解决这个问题，采取了编译时插桩的做法。
![QFix](./assets/32.png)
具体来说就是:
在所有类的构造函数中插入一段代码：`System.out.println(AntilazyLoad.class);` 
这样当安装apk的时候，classes.dex内的类都会引用一个在不相同dex中的AntilazyLoad类，这样就防止了类被打上`CLASS_ISPREVERIFIED`的标志了，只要没被打上这个标志的类都可以进行打补丁操作。

**优点：**
* 不需要考虑对dalvik虚拟机和art虚拟机做适配
* 代码是非侵入式的，对apk体积影响不大

**缺点：**
* 需要下次启动才会生效
* 性能损耗，即Dalvik平台为防止类打上CLASS_ISPREVERIFIED标志，插桩导致的性能损耗，Art平台由于地址偏移问题导致补丁包可能过大的问题


## Dex替换方案
如果QFix没有上面的缺陷，也可能就不会有微信的Tinker方案了。微信tinker采用将新旧apk做diff，得到diff.dex，然后下发diff.dex给终端手机，然后在运行时将diff.dex和手机中旧的classes.dex做合并，生成新的classes.dex，最后仍然采用上面的类加载方法，在启动时将新的classes.dex安排到dexElement数组的第一个元素。由于不需要代码插桩，性能比QFix有很大提高。

![compare](./assets/34.png)
![compare](./assets/35.png)
具体参考：<https://github.com/Tencent/tinker>

**优点：**
* 克服了QFix的性能缺陷
* 支持替换So库以及资源

**缺点：**
* 需要下次启动才会生效
* 实现复杂
Tinker性能痛点：
* Dex合并内存消耗在vm head上，容易OOM，最后导致合并失败。
* 如果本身app占用内存已经比较高，可能容易导致app本系统杀掉。

## Instant Run热插拔方案
在[Instant Run](./instantRun.md)文中可以了解Instant Run的大致原理。
美团Robust插件借鉴了Instant Run原理，对每个业务代码的每个函数都在编译打包阶段自动插入了一段代码，同时给每个class都增加了一个类型为ChangeQuickRedirect的静态成员，而在每个方法前都插入了使用changeQuickRedirect相关的逻辑，当 changeQuickRedirect不为null时，就会执行到accessDispatch从而替换掉之前老的逻辑，达到修复的目的。插入过程对业务开发是完全透明的。

```java
public long getIndex() {
      return 100;
}
 ```
 以上代码经过Robust框架注入后会被处理成:
```java
public static ChangeQuickRedirect changeQuickRedirect;
public long getIndex() {
     if(changeQuickRedirect != null) {
         if(PatchProxy.isSupport(new Object[0], this, changeQuickRedirect, false)) {
             return ((Long)PatchProxy.accessDispatch(new Object[0], this, changeQuickRedirect, false)).longValue();
         }
     }
     return 100L;
 }
```
当有补丁的时候changeQuickRedirect的值就不再是空，所以执行到需要热修的方法时就会走到补丁的方法实现而不是原逻辑达到修复目的


每个补丁包的结构
PatchesInfoImpl：补丁包说明类，可以获取所有补丁对象；每个对象包含被修复类名及该类对应的补丁类。
```java
public class PatchesInfoImpl implements PatchesInfo
 {
  public List getPatchedClassesInfo()
   {
     ArrayList localArrayList = new ArrayList();
     localArrayList.add(new  PatchedClassInfo("com.xxx.android.robustdemo.MainActivity", "com.bytedance.robust.patch.MainActivityPatchControl"));
     com.meituan.robust.utils.EnhancedRobustUtils.isThrowable = false;
     return localArrayList;
   }
  }
```

PatchesInfoImpl：补丁包说明类，可以获取所有补丁对象；每个对象包含被修复类名及该类对应的补丁类。
```java
public class MainActivityPatchControl implements ChangeQuickRedirect{
  ...
  //1.方法是否支持热修
  public boolean isSupport(String paramString, Object[] paramArrayOfObject)
  {
    Log.d("robust", "arrivied in isSupport " + paramString + " paramArrayOfObject  " + paramArrayOfObject);
    String str = paramString.split(":")[3];
    Log.d("robust", "in isSupport assemble method number  is  " + str);
    Log.d("robust", "arrivied in isSupport " + paramString + " paramArrayOfObject  " + paramArrayOfObject + " isSupport result is " + ":2:".contains(new StringBuffer().append(":").append(str).append(":").toString()));
    return ":2:".contains(":" + str + ":");
  }
  //2.调用补丁的热修逻辑
  public Object accessDispatch(String paramString, Object[] paramArrayOfObject)
  {
    for (;;)
    {
    try
      {
        Object localObject = new java/lang/StringBuffer;
        ((StringBuffer)localObject).<init>();
        if (!paramString.split(":")[2].equals("false")) {
            continue;
          }
        if (keyToValueRelation.get(paramArrayOfObject[(paramArrayOfObject.length - 1)]) != null) {
        continue;
      }
      localObject = new com/bytedance/robust/patch/MainActivityPatch;
      ((MainActivityPatch)localObject).<init>(paramArrayOfObject[(paramArrayOfObject.length - 1)]);
      keyToValueRelation.put(paramArrayOfObject[(paramArrayOfObject.length - 1)], null);
      paramArrayOfObject = (Object[])localObject;
      localObject = paramString.split(":")[3];
      paramString = new java/lang/StringBuffer;
      paramString.<init>();
       if (!"2".equals(localObject)) {
          continue;
        }
       paramString = paramArrayOfObject.RobustPublicgetShowText();
     }catch (Throwable paramString){ ...}
     return paramString;
     paramArrayOfObject = (MainActivityPatch)keyToValueRelation.get(paramArrayOfObject[(paramArrayOfObject.length - 1)]);
     continue;
     //具体实现逻辑在Patch中
     paramArrayOfObject = new MainActivityPatch(null);
   }
  }
 }
```
Patch：具体补丁方法的实现。该类中包含被修复类中需要热修的方法。
```java
public class MainActivityPatch
  {
   MainActivity originClass;
   public MainActivityPatch(Object paramObject)
   {
     this.originClass = ((MainActivity)paramObject);
    }
    //热修的方法具体实现
    private String getShowText()
    {
     Object localObject = getRealParameter(new Object[] { "Error Text" });
     localObject = (String)EnhancedRobustUtils.invokeReflectConstruct("java.lang.String", (Object[])localObject, new Class[] { String.class });
     localObject = getRealParameter(new Object[] { "Fixed Text" });
     return (String)EnhancedRobustUtils.invokeReflectConstruct("java.lang.String", (Object[])localObject, new Class[] { String.class });
     }
   }
```

优点：
几乎不会影响性能（方法调用，冷启动）
支持Android2.3-8.x版本
高兼容性（Robust只是在正常的使用DexClassLoader）、高稳定性，修复成功率高达99.9%
补丁实时生效，不需要重新启动
支持方法级别的修复，包括静态方法
支持增加方法和类
支持ProGuard的混淆、内联、优化等操作
缺点：
代码是侵入式的，会在原有的类中加入相关代码
so和资源的替换暂时不支持
会增大apk的体积，平均一个函数会比原来增加17.47个字节，10万个函数会增加1.67M。
会增加少量方法数，使用了Robust插件后，原来能被ProGuard内联的函数不能被内联了

## 底层方法替换方案
android虚拟机在加载一个类的时候会将类中方法解析成 ArtMethod 结构体，每一个Java方法在art中都对应着一个ArtMethod，ArtMethod记录了这个Java方法的所有信息，包括所属类、访问权限、代码执行入口地址（entrypoint)等等。通过把原方法对应的ArtMethod对象的entrypoint替换为目标方法的entrypoint，可以使得原方法被调用过程中取entrypoint的时候拿到的是目标方法的entry，进而直接跳转到目标方法的code段；从而达到Hook的目的。阿里AndFix就是基于这个原理来做热修复的。

ArtMethod结构体源码在"platform/art/runtime/art_method.h"中，并且基本每个android大版本，ArtMethod结构都有部分变化。

这是android 6.0版本中ArtMethod的定义：
```java
class ArtMethod {
  // Field order required by test "ValidateFieldOrderOfJavaCppUnionClasses". 
  // The class we are a part of. 
  GcRoot<mirror::Class> declaring_class_;

  // Short cuts to declaring_class_->dex_cache_ member for fast compiled code access. 
  GcRoot<mirror::PointerArray> dex_cache_resolved_methods_;

  // Short cuts to declaring_class_->dex_cache_ member for fast compiled code access. 
  GcRoot<mirror::ObjectArray<mirror::Class>> dex_cache_resolved_types_;

  // Access flags; low 16 bits are defined by spec. 
  uint32_t access_flags_;

  /* Dex file fields. The defining dex file is available via declaring_class_->dex_cache_ */

  // Offset to the CodeItem. 
  uint32_t dex_code_item_offset_;

  // Index into method_ids of the dex file associated with this method. 
  uint32_t dex_method_index_;

  /* End of dex file fields. */

  // Entry within a dispatch table for this method. For static/direct methods the index is into 
  // the declaringClass.directMethods, for virtual methods the vtable and for interface methods the 
  // ifTable. 
  uint32_t method_index_;

  // Fake padding field gets inserted here. 
  // Must be the last fields in the method. 
  // PACKED(4) is necessary for the correctness of 
  // RoundUp(OFFSETOF_MEMBER(ArtMethod, ptr_sized_fields_), pointer_size). 
  struct PACKED(4) PtrSizedFields {

    // Method dispatch from the interpreter invokes this pointer which may cause a bridge into 
    // compiled code. 
    void* entry_point_from_interpreter_;

    // Pointer to JNI function registered to this method, or a function to resolve the JNI function. 
    void* entry_point_from_jni_;

    // Method dispatch from quick compiled code invokes this pointer which may cause bridging into 
    // the interpreter. 
    void* entry_point_from_quick_compiled_code_;
  } ptr_sized_fields_;
}
```

这其中最重要的字段就是`entrypoint_from_interpreter_` 和 `entrypoint_from_quick_compiled_code_`2个指针了，从名字可以看出来，他们就是方法的执行入口。

我们知道，Java代码在Android中会被编译为Dex Code执行，ART虚拟机中可以采用解释模式或者AOT机器码模式执行。

解释模式，就是取出Dex Code，逐条解释执行。如果方法的调用者是以解释模式运行的，在调用这个方法时，就会取得这个方法的`entrypoint_from_interpreter_`指针，然后跳转过去执行。

而如果是AOT的方式，就会先预编译好Dex Code对应的机器码，然后运行期直接执行机器码就行了，不需要一条条地解释执行Dex Code。如果方法的调用者是以AOT机器码方式执行的，在调用这个方法时，就是跳转到`entry_point_from_quick_compiled_code_`执行。因此可以简单的认为，只要替换掉

这两个指针，就可以达到更改代码执行逻辑的目的。

那如何取到一个java方法对应的ArtMethod呢？

通过env -> FromReflectedMethod(srcMethodObj)，可以由Method对象得到这个方法对应的ArtMethod结构体的内存地址，然后可以把它强转为ArtMethod指针，从而对其所有成员进行修改。

但是上面也说了，每个android大版本，ArtMethod结构体的结构都有部分变化，这意味着里面的字段位置不是固定的，需要参考AOSP开源代码根据android版本来适配，另一方面《深入探索 Android 热修复技术原理》里也提到不同ROM厂商也可能会对ArtMethod定义进行修改(且不开源)，适配难度很大。

![method](./assets/62.png)

所以更保险的方法是对ArtMethod进行整体替换，而不用关心ArtMethod结构体中具体有什么，这就避免了大量的适配工作，这边是AndFix的升级版`Sophix`。要实现整体替换，就需要知道ArtMethod结构体的长度，但是这个长度会因为ArtMethod的不同版本或不同修改而不同。

有人可能想问用sizeof(ArtMethod)不就得到长度了吗？是的，但是前提是我们编译时使用的ArtMethod必须和虚拟机中的ArtMethod是同一个版本，这意味着我们需要在工程中针对不同的版本引入不用的art_method.h。

因为我们可以知道一个ArtMethod在内存上的起始地址，但是不知道结束地址，《深入探索 Android 热修复技术原理》里介绍了一个取巧的方法来获取ArtMethod的长度。就是取两个相邻ArtMethod的内存地址进行相减，从而间接得出ArtMethod的长度。这个方法的前提很明确，就是得构造2个java方法，且其对应的ArtMethod在内存上是相邻的，中间没有内存空隙。

`《深入探索 Android 热修复技术原理》`里介绍，参考ART虚拟机源码中对ArtMethod的排列逻辑，类的方法有direct_method 和 virtual_method之分，其中direct_method是包含那些static方法和不能继承的对象方法，而virtual_method就是那些可以继承的对象方法。很幸运的是，ArtMethod在内存中是线性排列的，就像下面这样.

![method](./assets/64.png)

我们在类中构造2个相邻的静态java空方法 f1 和 f2，方法名并不重要，重要的是他们是相邻的，且都是静态方法，因此他们都属于direct_method，而且在内存排列上也是相邻的。
```java
public class NativeStructsModel{
  final public static void f1(){}
  final public static void f2(){}
}
```

```java
size_t firMid = (size_t) env -> GetStaticMethodID(NativeStructsModelClazz, "f1", "()V");
size_t secMid = (size_t) env -> GetStaticMethodID(NativeStructsModelClazz, "f2", "()V");
size_t methodSize = secMid - firMid;
LOGD("########## methodSize = %d", methodSize);
```

使用Method Hook可以用来实现代码修复或者AOP，Sophix中将原ArtMethod丢弃，换成新的ArtMethod，这就无法实现AOP了。AndroidMethodHook基于Sophix的原理，用dexmaker动态生成类，将原方法保存下来，从而实现了AOP。

附测试代码：
```C++
#include <jni.h>
#include <string.h>
#include<android/log.h>

#define TAG "myDemo-jni" // 这个是自定义的LOG的标识
#define LOGD(...)  __android_log_print(ANDROID_LOG_DEBUG,TAG,__VA_ARGS__) // 定义LOGD类型

static const char* kClassMethodHookChar = "com/qylk/jetpack_demo/MethodHook";

static struct {
    jmethodID m1;
    jmethodID m2;
    size_t methodSize;
} methodHookClassInfo;


static long methodHook(JNIEnv* env, jclass type, jobject srcMethodObj, jobject destMethodObj) {
    int* srcMethod = reinterpret_cast<int*>(env -> FromReflectedMethod(srcMethodObj));
    int* destMethod = reinterpret_cast<int*>(env -> FromReflectedMethod(destMethodObj));
    int* backupMethod = new int[methodHookClassInfo.methodSize];

    memcpy(backupMethod, srcMethod, methodHookClassInfo.methodSize);
    memcpy(srcMethod, destMethod, methodHookClassInfo.methodSize);
    return reinterpret_cast<long>(backupMethod);
}

static jobject methodRestore(JNIEnv* env, jclass type, jobject srcMethod, jlong methodPtr) {
    int* backupMethod = reinterpret_cast<int*>(methodPtr);
    void* artMethodSrc = reinterpret_cast<void*>(env -> FromReflectedMethod(srcMethod));
    memcpy(artMethodSrc, backupMethod, methodHookClassInfo.methodSize);
    delete []backupMethod;
    return srcMethod;
}

static JNINativeMethod gMethods[] = {
        {
                "hook_native",
                "(Ljava/lang/reflect/Method;Ljava/lang/reflect/Method;)J",
                (void*)methodHook
        },
        {
                "restore_native",
                "(Ljava/lang/reflect/Method;J)Ljava/lang/reflect/Method;",
                (void*)methodRestore
        }
};

extern "C"
JNIEXPORT jint JNICALL
JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv *env = nullptr;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_4) != JNI_OK) {
        return JNI_FALSE;
    }
    jclass classEvaluateUtil = env->FindClass(kClassMethodHookChar);
    if(env -> RegisterNatives(classEvaluateUtil, gMethods, sizeof(gMethods)/ sizeof(gMethods[0])) < 0) {
        return JNI_FALSE;
    }
    methodHookClassInfo.m1 = env -> GetStaticMethodID(classEvaluateUtil, "m1", "()V");
    methodHookClassInfo.m2 = env -> GetStaticMethodID(classEvaluateUtil, "m2", "()V");
    methodHookClassInfo.methodSize = reinterpret_cast<size_t>(methodHookClassInfo.m2) - reinterpret_cast<size_t>(methodHookClassInfo.m1);
    LOGD("########## methodSize = %d", methodHookClassInfo.methodSize);
    return JNI_VERSION_1_4;
}
```