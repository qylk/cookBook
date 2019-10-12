当前hybird的趋势是：
h5  -> h5 + jsBridge –> h5 + jsBridge + 部分NativeUI组件 –> ReactNative 全量Native组件化。

对H5业务进行完整的Hybrid离线化。即：发版内置初始版本，后续可通过服务端增量更新，具体版本更新细则可以参照插件化增量更新机制，这样对于快速更新的业务有非常大的优势，可以在极短的时间内动态上线，而且可以做到多端统一。

如下为参照某Hybrid的目录结构：
```
webapp //根目录
├─
├─business1 //业务1
│  │  index.html //业务入口html资源
│  │  main.js //业务所有js资源打包
│  │
│  └─static //静态样式资源
│      ├─css 
│      ├─hybrid //存储业务定制化类Native Header图标
│      └─images
├─libs
│      libs.js //框架所有js资源打包
│
└─static //框架静态资源样式文件
    ├─css
    └─images
```