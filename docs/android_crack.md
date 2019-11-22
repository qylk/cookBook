# Android 逆向工程

## 工具
apktool: <https://github.com/iBotPeaches/Apktool/releases>

dex2jar: <https://github.com/pxb1988/dex2jar> 

jd-gui: <https://github.com/java-decompiler/jd-gui/releases>

## 反编译命令
```
java -jar apktool_2.4.0.jar d  xxx.apk
```

## 重编译命令
```
java -jar apktool_2.4.0.jar b <逆向工程目录>  -o build.apk
```

## APK重新签名 
```
jarsigner -keystore debug.jks xxx.apk <alias>
```

## dex2jar
```
sh d2j-dex2jar.sh classes.dex
```
生成classes-dex2jar.jar吗，可使用jd-gui浏览jar包里的代码。


# Smali语法
参考：[Smali 语法解析——Hello World](https://juejin.im/post/5c093fd751882535422e4f05)、[Smali语法介绍](https://blog.csdn.net/singwhatiwanna/article/details/19019547)