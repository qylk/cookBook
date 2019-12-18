# java代码打jar包

```
jar {ctxui}[vfm0Me] [jar-file] [manifest-file] [entry-point] [-C dir] files ...

选项包括：

    -c  创建新的归档文件

    -t  列出归档目录

    -x  解压缩已归档的指定（或所有）文件

    -u  更新现有的归档文件

    -v  在标准输出中生成详细输出

    -f  指定归档文件名

    -m  包含指定清单文件中的清单信息

    -e  为捆绑到可执行 jar 文件的独立应用程序，指定应用程序入口点

    -0  仅存储；不使用任何 ZIP 压缩

    -M  不创建条目的清单文件

    -i  为指定的 jar 文件生成索引信息

    -C  更改为指定的目录并包含其中的文件
```

1. 找到编译后的class文件目录
2. 执行打包 `jar cvfM mylib.jar <classDir>`，这会将所有class文件打包成mylib.jar