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


### 键盘弹起时，View高度没变
windowSoftInputMode='resize'
当decorView的`sysUiVisibility`设置`View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN`时，我们布局的根view的高度将保持不变(onSizeChanged不回调)，onLayout会回调，但参数不变。需要通过WindowVisibleDisplayFrame确定非键盘区域的真正大小。
```java
else if ((sysUiFl & View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN) != 0) {
                    pf.left = df.left = of.left = mRestrictedScreenLeft;
                    pf.top = df.top = of.top  = mRestrictedScreenTop;
                    pf.right = df.right = of.right = mRestrictedScreenLeft + mRestrictedScreenWidth;
                    pf.bottom = df.bottom = of.bottom = mRestrictedScreenTop
                            + mRestrictedScreenHeight;
                    if (adjust != SOFT_INPUT_ADJUST_RESIZE) {
                        cf.left = mDockLeft;
                        cf.top = mDockTop;
                        cf.right = mDockRight;
                        cf.bottom = mDockBottom;
                    } else {
                        cf.left = mContentLeft;
                        cf.top = mContentTop;
                        cf.right = mContentRight;
                        cf.bottom = mContentBottom;
                    }
                } 
```
注意：`WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS`，Added in API level 19，如果这个flag被设置，`View.SYSTEM_UI_FLAG_LAYOUT_STABLE` 和 `View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN` 这两个flag会被自动添加到sysUiVisibility中。
```java
private int getImpliedSystemUiVisibility(WindowManager.LayoutParams params) {
        int vis = 0;
        // Translucent decor window flags imply stable system ui visibility.
        if ((params.flags & WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS) != 0) {
            vis |= View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN;
        }
        if ((params.flags & WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION) != 0) {
            vis |= View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION;
        }
        return vis;
    }
```