# 概念：
* LightKV是基于Java NIO实现的轻量级，高性能，高可靠的key-value存储组件。

* LightKV包含AsyncKV和SyncKV两种模式，其中AsyncKV使用mmap文件内存映射，写入速度非常快，也不必关心数据何时写入文件；SyncKV模式类似于 SharePreferences，需要在数据修改后使用commit提交(比SharePreferences的apply提交方式要慢很多)。

* LightKV使用SparseArray作为内存级缓存的数据结构，效率上比传统的HashMap要高。

# 参考资料：

[https://juejin.im/post/5b2faa3ff265da599731d64a](https://juejin.im/post/5b2faa3ff265da599731d64a)

源码：

[https://github.com/No89757/LightKV](https://github.com/No89757/LightKV)