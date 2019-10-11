### 1、gradle打包脚本中配置flavor  

每一个渠道重走一遍打包流程，缺点：慢，适用于渠道少的场景或者不同渠道间有不同资源(如页面、图片等)的场景。  
<br>

### 2、基于v1签名的zip摘要渠道写入方式  

利用zip文件可以添加comment（摘要）的数据结构特点，在zip文件的末尾的摘要区域写入渠道信息，而不用重新解压zip文件，且v1签名机制下不需要重新签名。  

优点：快  
<br>

### 3、基于v1签名的META-INF渠道写入方式

在apk文件(zip文件)的META-INF目录下写入名称为某个渠道字符串的空白文件，原理是基于v1签名机制下，META-INF中添加一个空文件不需要重新签名。

美团较早之前的渠道写入使用该方式

优点：快  
<br>

### 4、基于v2签名的渠道写入方式

基于android v2签名，在apk(zip格式)的APK Signing Block区块中加入渠道信息，且apk不需要重新签名。优点：快

原理可参考：[https://www.jianshu.com/p/](https://www.jianshu.com/p/8d4396ce231f)、 [https://www.jianshu.com/p/3fd8542f709a](https://www.jianshu.com/p/3fd8542f709a)
<br>
美团大部分app使用该方式打渠道包。

工具开源地址：[https://github.com/Meituan-Dianping/walle](https://github.com/Meituan-Dianping/walle)

<br>

参考：[https://zhuanlan.zhihu.com/p/26674427](https://zhuanlan.zhihu.com/p/26674427)