### 1. Android 7.0 Notification Sound Issue
通知铃声Uri如下，格式为FileProvider Uri
```
content://xxxxxxxx/yyyy.mp3
```
**原因**：

Notification所在进程"com.android.systemui"无访问权限，导致设置的MP3无法播放

**解决**：

1. 对于xxxxxxxx为自身应用的authorities时，添加一下代码，可授权访问权限。
```
grantUriPermission("com.android.systemui", sound,Intent.FLAG_GRANT_READ_URI_PERMISSION);
```

2. 对于xxxxxxxx为其他应用的authorities时，需要将xxxxxxxx改为自身应用的authorities，否则无法授权。

3. 对于自身应用的资源文件, 可使用使用android.resource协议，无需授权。
```
Uri uri = Uri.parse("android.resource://“+ context.getPackageName() + "/raw/test.mp3")
```
> 备注：从 android 7.0开始，如果应用的targetSdkVersion>=24，则不同应用之间传递uri不能再使用file://协议，否则将抛出FileUriExposedException