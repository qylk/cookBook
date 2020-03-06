# MultiDex
## MultiDex存在的问题
我们经常说的MultiDex，可以分成运行时和编译时两个部分：  
* 编译时的分包机制，将app中的class以某种策略分散在多个dex中，目的是减少为了第一个dex也就是main dex中包含的class
* 运行时： app启动时，虚拟机只加载main dex中的class。app启动以后，使用Multidex.install API，通过反射修改ClassLoader中的dexElements加载其他dex

MultiDex机制的出现本身是为了避免出现app 65535问题的出现，但随着业务逻辑的增长，以及不合理的模块划分，可能会导致main dex的方法数也超出了65535，这就导致了main dex capacity exceeded异常。

此外，Multidex的接入额外还会对app的启动性能造成影响。Multidex在install时需要加载dex，首次启动时还需要做odex的转换，而这些都是在ui主线程中完成。
根据 Carlos Sessa的测试，启用multidex后，4.4或以下的设备，app的启动时间平均会增加15%，更严重的情况，甚至在启动时候会出现了黑屏。

因此目前部分app采取的策略是，放弃掉Multidex的，而转为插件化的架构。通过将非核心模块的lazy load，来达到启动速度的优化，但我们需要明确的是，并不是所有app都适合插件化架构，为了实现启动加速将本耦合的业务逻辑硬生生拆解其实是本末倒置。

## 解决方案  
### Multidex异步化

在Android的性能优化中，最常见的思路就是异步化，减少UI线程的工作。在应用的交互层面上，app启动时，几乎所有app都会有一个SplashActivity。在界面上展示欢迎页，在后台进行初始化的业务逻辑。这就给我们一个启发，我们可以将系统的初始化逻辑，延迟到我们的业务初始化时间点上。

更加具体的方式是，我们可以将Multidex.install这个操作异步化，保证主线程的正常进行，待dex加载完成后通知SplashActivity跳转到真正的业务主界面。

对MultiDex的加载进行异步化之后，我们还可以进行第二步，main dex大小的精简。

### Main Dex精简
我们先了解一下MultiDex分包的原理，Multidex会在入口Application的attachBaseContext，加载second dex，因此multidex分包的基本原则是：保证app启动需要的class放置在main dex上。在android gradle 1.5之后，multidex都通过一个MultidexTransform完成，分包过程可以分为三步：

1. 生成manifest_keep.txt
MutidexTransform会解析出AndroidManifest.xml中所有的组件类：包括Activity、Service、Receiver以及ContentProvider，这些类将和Application入口类一起放在build/intermediates/multi-dex/{flavor}/{buildType}/manifest_keep.txt文件中

2. 生成maindexlist.txt文件
查找manifest_keep.txt中所有类的直接引用类，具体的方式是遍历类的所有字段以及方法，查看方法的参数和返回值的类型，将其放保存在maindexlist.txt

3. 生成main dex
将maindexlist.txt文件包含的所有class编译进main dex

从上面的分析中，我们可以确定的是，MultiDex的分包机制并不严密：

MultiDex将AndroidManifest.xml中的所有组件都包含在了manifest_keep.txt。但app在首次启动时，并不需要加载所有的组件，而只是需要入口的activity，供其他app访问的service、contentprovider以及注册获取系统通知的receiver。MainDex中过多的组件信息反而可能导致了app启动过慢。

MultidexTransform只查找了manifest_keep.txt中类的直接引用类，间接引用类并没有出现在maindex中，特殊情况下，会出现NoClassDefFoundError的异常，这时候开发者需要自行将需要的class添加到maindexlist.txt

针对这两个缺陷，我们的优化思路是MultiDex的分包流程进行改进：

使用SAX自行解析AndroidMainfest.xml，抽取出组件信息，将原始的Manifest_keep.txt内容替换掉，去除启动不需要的Activity组件，保证启动加载的类最小。

在gradle中添加multiDexExt扩展块，通过指定类名或通配符来设置必须编译在MainDex中类，在扩展块中指定的类都会被添加到maindexlist.txt文件汇中。

```groovy
multiDexExt {
    keepClasses += 'android.support.v7.app.AppCompatActivity'
    keepClasses += 'android.support.v7.app.AppCompatDelegate'
    keepClasses += 'android.support.v7.app.**'
}
```
额外需要提的一个细节是，为了保证以上精简生效。我们还需要开启dx工具的minimal-main-dex参数：


这个参数可以保证MainDex只包含maindexlist.txt文件中指定的类。但在gradle1.5到2.2之间，这个参数被默认关闭的，直到gradle2.2之后，dx的minimal-main-dex才重新开放给了开发者。

最后
在main dex的分包过程中，maindex只包含了组件以及直接引用类。通过我们的优化进一步减少了maindex的大小，因此也增大了NoClassDefFoundError的异常的可能，使用以上的优化思路做好测试，一旦发现启动失败，使用multiDexExt重新添加缺失的类