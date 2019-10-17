tcpdump用于手机网络的抓包工具
* 需要手机root

下载地址：<http://www.strazzere.com/android/tcpdump>

步骤：
1. 下载tcpdump
2. 上传到手机 /data/local/tmp目录下
3. 修改权限 chmod +x tcpdump
4. 启动tcpdump抓包
```shell
tcpdump -p -vv -s 0 -w /sdcard/1.pcap
```
```shell
tcpdump -p -vv -s 0 'port 1111' -w /sdcard/1.pcap
```
5. Ctrl+C结束抓包
6. 导出抓包数据文件
6. 使用Charles或者WireShark分析抓包数据

更多tcpdump命令参数请参考：
<https://blog.csdn.net/hzhsan/article/details/43445787>